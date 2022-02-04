---
title: "スケジュール実行で Aurora DB クラスタのレプリカ台数を変える"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","aurora", "autoscaling"]
published: true
---


# やりたいこと

週次のイベントのために曜日時刻指定で Aurora DB クラスタ のレプリカ台数を増減させたい。

# やったこと

反省も兼ねて、自分がやったことを時系列順に書いていきます。

## マネージメントコンソールを確認

レプリカ台数を増減させたい Aurora DB クラスタ の設定を確認したところ、Aurora Auroscaling ポリシーの設定箇所を発見しました。

![](/images/aurora-autoscaling-for-scheduled-execution-with-fixed-number-of-replicas/management_console.png =500x)

この表示で Aurora Auto Scaling ポリシーの存在、レプリカのCPU使用率もしくは接続数の平均を使ったスケールアウト・スケールイン構成が可能なことを知りました。

Aurora Autoscaling ポリシーを使ってやりたいことを実現するなら、閾値設定で絶対に達することのない値(接続数10000等)を設定し、EventBridgeを使ったCronスケジュールでAurora Autoscalingポリシーの台数をLambdaで変更すればよいかと思っていました。

## AWS CLIのドキュメントを確認

Lambdaでレプリカ台数制御する際に使用可能なAPIが存在するか確認するために、AWS CLIのドキュメントを確認しました。
これは確認しようと思ってこのドキュメントにたどり着いたわけではなく、`Aurora Autoscaling aws cli`等でGoogle検索して見つけたページでした。

https://awscli.amazonaws.com/v2/documentation/api/latest/reference/application-autoscaling/index.html#cli-aws-application-autoscaling

これを確認すると、以下のようなことがわかりました。

- application-autoscaling でAuroraに限らず他のAWSサービスのリソースの自動スケーリングを設定可能である。
- リンク内の Available Commands より、スケーリングポリシー(scaling-policy)だけでなく、スケーラブルターゲット(scalable-target)・スケジュールドアクション(scheduled-actions)というリソースが存在する。

これから、Auto Scaling ポリシーを使わなくても、スケジュールドアクションを設定できれば容易にやりたいことができそうだとわかりました。

そして、公式情報では見つけられなかったのですが、スケーラブルターゲットやスケジュールドアクションの設定値をマネージメントコンソール上で確認できないことがわかってきました。
少なくとも2022年2月3日現在、東京リージョンでこの構成をマネージメントコンソールで確認することができず、API経由での操作や確認が必要となります。

今回詳細には触れないですが、[AWS Documentation](https://docs.aws.amazon.com/)のページには[Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/)へのリンクがあり、Auto Scaling には Amazon EC2 Auto Scaling・Application Auto Scaling・AWS Auto Scaling の3種類があることがわかりました。
こちらから調べれば、Auto Scalingサービスの全体像の把握やどのサービスを使うべきかも知れ、今回知った情報にも早くたどり着けそうでした。

## 試行錯誤

テスト用の Aurora DB クラスタで、スケジュールドアクションを作成するために、以下のようなコマンドを実行しましたが、エラーが返ってきました。

```
aws application-autoscaling put-scheduled-action --service-namespace rds \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --resource-id cluster:test-cluster \
    --scheduled-action-name test-action \
    --schedule "cron(0 0 ? * MON *)" \
    --scalable-target-action MinCapacity=1,MaxCapacity=1

An error occurred (ObjectNotFoundException) when calling the PutScheduledAction operation: No scalable target registered for service namespace: rds, resource ID: cluster:test-cluster, scalable dimension: rds:cluster:ReadReplicaCount
```

スケーラブルターゲットを登録しないと、スケジュールドアクションを設定できないので、以下の順番でコマンド実行しました。

```
aws application-autoscaling register-scalable-target \
    --service-namespace rds \
    --resource-id cluster:test-cluster \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --min-capacity 1 --max-capacity 1

aws application-autoscaling put-scheduled-action \
    --service-namespace rds \
    --scalable-dimension rds:cluster:ReadReplicaCount \
    --resource-id cluster:test-cluster \
    --scheduled-action-name test-action \
    --schedule "cron(0 0 ? * MON *)" \
    --scalable-target-action MinCapacity=1,MaxCapacity=1
```
ちなみに、レプリカの台数を固定したかったため、レプリカ台数の最小値と最大値の値を同じ値としていました。

レプリカ台数を増減させる試行錯誤を行う中で、Aurora Auto Scaling がレプリカ台数をカウントする際に、Aurora Auto Scaling が作成していない(例えば、手動作成した)レプリカを含めてクラスタ管理下のレプリカを数えていることがわかりました。
また、[ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.Integrating.AutoScaling.html#:~:text=Aurora%20Auto%20Scaling%20%E3%81%A7%E3%81%AF%E3%80%81%E8%87%AA%E8%BA%AB%E3%81%8C%E4%BD%9C%E6%88%90%E3%81%97%E3%81%9F%20Aurora%20%E3%83%AC%E3%83%97%E3%83%AA%E3%82%AB%E3%81%AE%E3%81%BF%E5%89%8A%E9%99%A4%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%E3%80%82)にも記載がありますが、Aurora Auto Scaling が作成したインスタンスはスケールインで削除されますが、手動作成したインスタンスは削除されないことを確認しました。

## 構成検討

業務運用と Aurora DB クラスタの既存の自動運用スケジュール(スケールアップ/スケールダウンとそのためのフェイルオーバー)に加えて、今回の要件に合わせて Aurora Auto Scaling の実行スケジュールを決めました。
今回は、AWS CLI で設定しましたが、CloudFormation や AWS CDK などで構成管理するのがベストな方法だと思います。

## 動作確認と対応

設定し、実際に動作確認したところ、当初決めたスケジュールだと想定通りの台数にスケールアウト/スケールインが発生しないことがありました。
これは、Aurora Auto Scaling がスケジュール実行されるタイミングで、Aurora DB クラスタを構成するインスタンス全てが利用可能な状態になっていないことが原因でした。具体的には、スケジュール実行される直前に実行していたレプリカ1台のスケールアップが完了してない状態となっていました。

このときのスケーラブルターゲットの状態を `aws application-autoscaling describe-scalable-targets`コマンドで確認すると、レプリカ台数設定は Auto Scaling によって更新されていました。これより、スケーラブルターゲットの変更タイミングでクラスタを構成する全てのインスタンスが利用可能な状態でないと、スケーラブルターゲットに加えた変更どおりのレプリカ台数とならないと理解しました。
(私が確認した挙動だと[Aurora レプリカでの Amazon Aurora Auto Scaling の使用](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.Integrating.AutoScaling.html#:~:text=Aurora%20%E3%83%AC%E3%83%97%E3%83%AA%E3%82%AB%E3%81%AE%E3%81%84%E3%81%9A%E3%82%8C%E3%81%8B%E3%81%8C%E5%88%A9%E7%94%A8%E5%8F%AF%E8%83%BD%E4%BB%A5%E5%A4%96%E3%81%AE%E7%8A%B6%E6%85%8B%E3%81%AB%E3%81%82%E3%82%8B%E5%A0%B4%E5%90%88%E3%80%81Aurora%20Auto%20Scaling%20%E3%81%AF%20DB%20%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%83%BC%E5%85%A8%E4%BD%93%E3%81%8C%E3%82%B9%E3%82%B1%E3%83%BC%E3%83%AA%E3%83%B3%E3%82%B0%E3%81%AB%E5%88%A9%E7%94%A8%E5%8F%AF%E8%83%BD%E3%81%AB%E3%81%AA%E3%82%8B%E3%81%BE%E3%81%A7%E5%BE%85%E6%A9%9F%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)に記載されているような待機状態にもなりませんでした。)

今回は、運用スケジュール全体に余裕をもたせていたこともあり、スケールアップのスケジュールを早め、Auto Scaling 実行時に全てのインスタンスが利用可能な状態となるようにすることで対応しました。
もし運用スケジュールに余裕がない場合は、スケールアップの処理が完了したことを示すイベントを受けるか、処理全体をStepFunctionsで制御し、なってほしい状態になるまでループさせるような処理の実装も視野にいれてもよいかもしれないです。

別の観点ですが、Auto Scaling によって作成された Aurora DB インスタンスのマイナーバージョンアップが自動でされない設定にしなければとならないと考えていました。というのも、Auto Scaling が作成したインスタンスのマイナーバージョン自動アップグレードが有効になっていたからです。
ただ、これに関しては、[Aurora MySQL DB クラスターのマイナーバージョンまたはパッチレベルのアップグレード](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Updates.Patching.html#:~:text=%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%83%BC%E5%86%85%E3%81%AE%E3%81%84%E3%81%9A%E3%82%8C%E3%81%8B%E3%81%AE%20DB%20%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%81%A7%E3%83%9E%E3%82%A4%E3%83%8A%E3%83%BC%E3%83%90%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3%E8%87%AA%E5%8B%95%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89%E3%81%AE%E8%A8%AD%E5%AE%9A%E3%81%8C%E3%82%AA%E3%83%B3%E3%81%AB%E3%81%AA%E3%81%A3%E3%81%A6%E3%81%84%E3%81%AA%E3%81%84%E5%A0%B4%E5%90%88%E3%80%81Aurora%20%E3%81%AF%E3%81%9D%E3%81%AE%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%83%BC%E5%86%85%E3%81%AE%E3%81%84%E3%81%9A%E3%82%8C%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%82%82%E8%87%AA%E5%8B%95%E7%9A%84%E3%81%AB%E3%81%AF%E3%82%A2%E3%83%83%E3%83%97%E3%82%B0%E3%83%AC%E3%83%BC%E3%83%89%E3%81%97%E3%81%BE%E3%81%9B%E3%82%93%E3%80%82%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%83%BC%E5%86%85%E3%81%AE%E3%81%99%E3%81%B9%E3%81%A6%E3%81%AE%20DB%20%E3%82%A4%E3%83%B3%E3%82%B9%E3%82%BF%E3%83%B3%E3%82%B9%E3%81%A7%E8%A8%AD%E5%AE%9A%E3%81%8C%E4%B8%80%E8%B2%AB%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E3%81%93%E3%81%A8%E3%82%92%E7%A2%BA%E8%AA%8D%E3%81%97%E3%81%A6%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82)にある通り、もともと作成していたインスタンスの設定で無効となっていれば、問題なさそうなことがわかりました。

# まとめ

- [Application Auto Scaling API](https://docs.aws.amazon.com/autoscaling/application/APIReference/)を使って、以下の構成としました。
    - [RegisterScalableTarget](https://docs.aws.amazon.com/autoscaling/application/APIReference/API_RegisterScalableTarget.html)でAurora DB クラスタにスケーラブルターゲットを登録します。この設定を施したタイミングで指定した台数にレプリカ台数が調整されます。
    - [PutScheduledAction](https://docs.aws.amazon.com/autoscaling/application/APIReference/API_PutScheduledAction.html)で、上記で登録したスケーラブルターゲットに対する設定変更のスケジュール実行をスケジュールドアクションとして登録します。

- 利用のための注意点として、以下3点がありました。
    - Aurora Auto Scaling がレプリカ台数をカウントする際に、Aurora Auto Scaling が作成していない(例えば、手動作成した)レプリカを含めてクラスタ管理下のレプリカを数えます。[ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Aurora.Integrating.AutoScaling.html#:~:text=Aurora%20Auto%20Scaling%20%E3%81%A7%E3%81%AF%E3%80%81%E8%87%AA%E8%BA%AB%E3%81%8C%E4%BD%9C%E6%88%90%E3%81%97%E3%81%9F%20Aurora%20%E3%83%AC%E3%83%97%E3%83%AA%E3%82%AB%E3%81%AE%E3%81%BF%E5%89%8A%E9%99%A4%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%E3%80%82)にも記載がありますが、Aurora Auto Scaling が作成したインスタンスはスケールインで削除されますが、手動作成したインスタンスは削除されませんでした。
    - クラスタ内の全てのレプリカが利用可能な状態になっていない場合、スケジュールドアクションで設定した日時になってもAuto Scaling は動作しませんでした。
    - Aurora DB クラスタの仕様として、マイナーバージョン自動アップグレードは、クラスタ内の全てのインスタンスで有効になっていないと実行されないです。

- 調べる際、[AWS Documentation](https://docs.aws.amazon.com/index.html)から調査を開始すべきでした。

