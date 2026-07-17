---
title: "CloudFront VPC Origin障害をどう切り分けて復旧したか"
emoji: "🚨"
type: "tech"
topics: ["AWS", "CloudFront", "VPCOrigin", "ALB", "障害対応"]
publication_name: nextbeat
published: false
---

# はじめに

2026年7月16日、AWSのCloudFront VPC Originに障害が発生し、弊社の複数プロダクトが一斉にアクセス不能となりました。

https://health.aws.amazon.com/health/status

障害発生当初はAWS側から障害情報が公開されておらず、「これは本当にAWSの障害なのか？それとも自分たちの構成の問題なのか？」がわからない状態からの切り分けとなりました。

今回はその際に実際に行った障害の切り分け手順と、復旧までの流れを紹介します。VPC Originを使用した構成を採用している方や、これから採用を検討している方の参考になれば幸いです。

:::message
記事内のリソースID、ドメイン、IPアドレスなどはマスクもしくは置き換えを行っています。また、記事内の時刻は全てJSTです。

記事内のスクリーンショットに関しては、調査当時はスクショを残す余裕がなかったため、弊社の勉強用AWSアカウントで同様の環境を構築して取得したものです。当時の実際の出力とは細部が異なる場合があります。
:::

# VPC Originとは

まず、今回の障害の主役となったVPC Originについて簡単に説明しておきましょう。

VPC Origin (CloudFront Virtual Private Cloud origins) は2024年11月に発表されたCloudFrontの機能で、プライベートサブネットにあるALBやEC2などをCloudFrontのオリジンとして直接指定できるようになるものです。

https://aws.amazon.com/jp/blogs/news/introducing-amazon-cloudfront-vpc-origins-enhanced-security-and-streamlined-operations-for-your-applications/

従来の構成ではCloudFrontのオリジンにALBを指定する場合、ALBをパブリックサブネットに配置してインターネットからアクセス可能にしておく必要がありました。VPC Originを使用することでALBを完全にプライベートなまま保つことができるため、CloudFrontを経由しないアクセスを構成レベルで遮断でき、セキュリティを向上させることができます。

VPC Originを作成すると、CloudFrontは顧客のVPC内にマネージドENI (`InterfaceType: cloudfront_managed`) を自動作成します。CloudFrontエッジからのトラフィックはこのENIを通ってVPCに入り、内部ALBへ届きます。

```
CloudFrontエッジ →（AWS内部網）→ [cloudfront_managed ENI] → 内部ALB → ECS
```

弊社ではこのVPC Originを採用しており、複数のプロダクトが「CloudFront → VPC Origin → 内部ALB → ECS」という構成で動いていました。この前提を頭に入れた上で、ここからは障害当日の切り分けの流れを見ていきましょう。

# 障害発生

<!-- TODO: 障害を検知した時刻を記載 -->

発端は、全プロダクトのアラートが一斉に鳴り始めたことでした。確認してみると、特定のプロダクトだけではなく全てのプロダクトへアクセスできなくなっていました。

全プロダクトが同時にダウンしたと聞くと真っ先にインフラの全面障害を疑いたくなりますが、各リソースを確認すると不思議な状態でした。

- ECS、Aurora、OpenSearchは正常に稼働しており、負荷も高くない
- 攻撃的なアクセスも確認できない
- しかしユーザーからはどのプロダクトにもアクセスできない

バックエンドは全て健康なのに入口だけが死んでいる。となると疑うべきは入口、つまりCloudFrontです。

「CloudFrontの障害かな？」

そう考えてXやAWS Health Dashboardの情報を拾い集めるものの、障害報告が全く出てきません。CloudFrontほどの規模のサービスで全域障害が起きていれば、SNSはお祭り状態になっているはずです。それがないということは、次の2つの可能性が考えられます。

- CloudFront全体ではなく、その背後にある特定の機能の障害
- もしくは自社の構成に問題がある

どちらにせよ「AWSの障害報告を待っていても直らない」可能性があるため、切り分けを進めることにしました。

# 問題の切り分け

## VPC OriginのENIを確認する

まずは自社構成の問題かどうかを確認するため、VPC OriginがVPC内に作成するマネージドENIの状態を確認しました。

```shell
aws ec2 describe-network-interfaces --region ap-northeast-1 \
    --filters "Name=vpc-id,Values=vpc-xxxxxxxxxxxxx" \
    --query "NetworkInterfaces[?contains(Description,'CloudFront') ||
  contains(InterfaceType,'cloudfront')].{Desc:Description,Type:InterfaceType,AZ:AvailabilityZone,Status:Status,IP:PrivateIpAddress,Subnet:SubnetId}" \
    --output table
```

このコマンドで確認したかったのは以下の2点です。

1. **ENIが存在しin-useか** — 誰かが誤って削除していたり`failed`状態であれば構成レベルの問題であり、自分たちで直せる可能性があります
2. **どのAZ・サブネットにあるか** — 当時はAZ障害説も残っていたため、ENIが片方のAZにしかなければ「そのAZが死んだ」で説明がつきます

出力結果は以下のようになります。

![DescribeNetworkInterfacesの出力結果](/images/aws-vpc-origin-outage-troubleshooting/describe-network-interfaces.png)

結果はというと、ENIは`cloudfront_managed`のタイプで全て`in-use`で存在しており、`ap-northeast-1a`と`ap-northeast-1c`の両AZに配置されておりAZの偏りもありませんでした。

つまり、自アカウントから見えるリソース (ALB・ターゲット・ENI・セキュリティグループ・設定変更履歴) は全てシロ。それでもトラフィックが届かないということは、故障箇所はENIの見た目からはわからないAWS内部のデータプレーン (エッジ→ENI間の経路制御) にある可能性が出てきました。

## CloudFront経由でアクセスできるものを確認する

次に、CloudFrontを経由するアクセスのうち「何が通って何が通らないのか」を整理し直しました。

| 経路 | 結果 |
|------|------|
| CloudFront経由のアプリケーション接続 (VPC Origin → 内部ALB) | ❌ |
| CloudFront経由のS3接続 (静的サイト) | ⭕️ |
| CloudFront経由のS3 + API接続 (SPA) | ❌ (jsやhtmlは取得できるがAPIがダメ) |

面白いのは3つ目で、アプリケーションのjsファイルやhtmlは取得できるのに、API経由でのデータ取得だけが失敗していました。

S3をオリジンとする配信は正常で、VPC Originを経由する通信だけが死んでいる。この時点で「CloudFront自体の問題ではなく、その後ろが原因」という仮説が濃厚になってきました。

## ログでどこまでリクエストが届いているかを確認する

仮説を検証するため、GrafanaでCloudFrontとALBのログを確認しました。

![GrafanaでのCloudFrontとALBのログ確認](/images/aws-vpc-origin-outage-troubleshooting/grafana-cloudfront-alb-logs.png)

- CloudFrontのログ → ⭕️ リクエストが記録されている
- ALBのログ → ❌ `No data` (リクエストが届いていない)

リクエストはCloudFrontまでは到達しているが、ALBには届いていないことが確認できました。

ここでALBを原因と仮定してALB側の調査を進めました。

## ALBのターゲットヘルスを確認する

ALBが原因だと仮定した場合、考えられる故障パターンは次の3つです。

- (a) アプリ (ECSタスク) が落ちている
- (b) ALB→ターゲット間が切れている (セキュリティグループ・ネットワーク)
- (c) ALBより手前で切れている

まずは(a)と(b)を確認するため、内部ALBにぶら下がる全ターゲットグループのヘルス状態を確認しました。

```shell
aws elbv2 describe-target-groups --region ap-northeast-1 \
    --load-balancer-arn arn:aws:elasticloadbalancing:ap-northeast-1:xxxxxxxxxxxx:loadbalancer/app/xxxxxxxxxx/xxxxxxxxxx \
    --query "TargetGroups[].TargetGroupArn" --output text | tr '\t' '\n' | while read tg; do
      echo "=== $tg"
      aws elbv2 describe-target-health --region ap-northeast-1 --target-group-arn "$tg" \
        --query "TargetHealthDescriptions[].{Id:Target.Id,AZ:Target.AvailabilityZone,State:TargetHealth.State,Reason:TargetHealth.Reason,Desc:TargetHealth.Description}" \
        --output table
  done
```

`describe-target-groups`で内部ALBにぶら下がる全ターゲットグループを列挙し、それぞれに対して`describe-target-health`で登録されているターゲット (ECSタスクのIP) のヘルス状態を取得しています。

出力結果は以下のようになります。

![DescribeTargetHealthの出力結果](/images/aws-vpc-origin-outage-troubleshooting/describe-target-health.png)

結果、全てのターゲットが`healthy`でした。(a)と(b)はシロということがわかりました。

## ALBに直接アクセスして確認する

残るは(c)「ALBより手前で切れている」ですが、これを確定させるには「ALB自体は正常に応答できる」ことを証明する必要があります。

そこで、VPC内の踏み台サーバーからALBに直接リクエストを送って確認することにしました。

まず、ALBのセキュリティグループに踏み台からの443アクセスを一時的に許可し、SSMセッションマネージャーで踏み台に入ります。

```shell
aws ssm start-session --target i-xxxxxxxxxxx --region=ap-northeast-1
```

踏み台からALBのプライベートIPに対して、Hostヘッダーを本番ドメインに解決させた状態でリクエストを送ります。

```shell
curl -sv --resolve app.example.com:443:10.0.xx.xx https://app.example.com/ -o /dev/null 2>&1 | grep "< HTTP"
< HTTP/2 200
```

**HTTP/2 200**。

確認したところ200が帰ってきたのでALBは生きているということがわかりました。

## 原因の確定

ここまでの結果を整理してみましょう。

- CloudFrontは正常 (S3オリジンの配信はできており、アクセスログも記録されている)
- ALBも正常 (ターゲットは全てhealthyで、VPC内からのリクエストには200を返す)
- しかしCloudFrontからALBへのリクエストだけが届かない

ということは、CloudFrontとALBの間で落ちている。間にいるのは……

```
CloudFrontエッジ →（AWS内部網）→ [cloudfront_managed ENI] → 内部ALB → ECS
       ⭕️              ❌ ここ            (見た目は正常)        ⭕️        ⭕️
```

VPC Originです。

自アカウントから見える全てのリソースはシロで、消去法によりVPC Origin (のAWS内部データプレーン) が原因と仮説を立てました。

# 復旧対応

原因の仮説が立ったところで、次は復旧です。とはいえVPC OriginのデータプレーンはAWSのマネージド領域であり、自分たちで直すことはできません。復旧を待つか、VPC Originを経路から外すかの2択です。

弊社はVPC Origin導入前にPublic ALBを構成を採用していたので急遽この構成を復元し、CloudFrontのオリジンをVPC OriginからPublic ALBへ切り替えることにしました。

```
[障害時]  CloudFront → VPC Origin → 内部ALB → ECS
[切り戻し] CloudFront → Public ALB → ECS
```

まず1つのプロダクトで切り戻しを実施し、復旧することを確認。これでVPC Originが原因であることが確定したため、残りのプロダクトへ横展開を開始しました。

その結果、早いサービスでは18時前には復旧を開始することができました。

なお、AWS側の完全復旧は22時前までかかっていたようなので、障害報告を待たずに自分たちで切り分けて切り戻す判断をしたのは正解だったと思います。

# 学び

今回の障害対応を通して、いくつかの学びがありました。

**復旧優先順位を事前に決めておく必要がある**

耐障害性を高めることも大事ですが、複数プロダクトが同時に落ちた場合に「会社としてどのプロダクトを第一に復旧させるべきか」を判断できるようにしておく必要があると感じました。切り戻し作業は1プロダクトずつ行うため、優先順位が決まっていないと合意形成に時間を取られてしまいます。

**PrivateからPublicへ変更する判断基準を持っておく**

VPC Originからの切り戻しは、言い換えると「セキュリティのために閉じた経路を、可用性のために一時的に開ける」判断です。この判断を障害の混乱の中で個人がその場で下すのは難しいため、判断基準や判断者を事前に決めておく必要があると感じました。

**マネージドサービスの障害は「見た目は正常」なことがある**

今回、VPC OriginのENIはAWS CLIから見る限り完全に正常でした。マネージドサービスの内部データプレーンの障害は利用者側から観測できないため、「リソースの見た目が正常であること」と「トラフィックが流れること」は別物として切り分ける必要があります。今回だと踏み台からALBへ直接リクエストを送った確認がまさにそれで、経路の両端が正常であることを証明できたことで、間にいるVPC Originを消去法で特定することができました。

# まとめ

今回はCloudFront VPC Originの障害について、切り分けの手順と復旧までの流れを紹介しました。

改めて切り分けの流れを整理すると以下のようになります。

1. バックエンドは正常なのに全プロダクトが落ちている → 入口 (CloudFront) を疑う
2. 障害報告が出てこない → 待っていても直らない可能性を考慮して自分たちで切り分け
3. VPC OriginのENIを確認 → 構成レベルはシロ
4. CloudFront経由のアクセスを整理 → S3は⭕️、VPC Origin経由だけ❌
5. ログを確認 → リクエストはCloudFrontまで届き、ALBには届いていない
6. ターゲットヘルスを確認 → 全てhealthy
7. 踏み台からALBへ直接リクエスト → 200が返る
8. 消去法でVPC Originが原因と確定 → Public ALBへ切り戻して復旧

障害対応はどうしても場当たり的になりがちですが、「経路のどこまでが正常か」を両端から順番に証明していくことで、利用者からは見えないマネージド領域の障害でも原因箇所を特定することができました。

VPC Originのようなまだ新しい機能は障害情報が出てくるのも遅い場合があるため、同じ構成を採用している方は「VPC Originを外した経路に切り替えられるか」を事前に確認しておくことをおすすめします。

最後に、障害対応に協力してくれたチームのみなさん、ありがとうございました。
