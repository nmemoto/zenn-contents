---
title: "EventBridgeでS3イベント通知を使う構成をCDKで作ってみた"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "AWS CDK", "eventbridge"]
published: true
---

# きっかけ

同僚との会話の中で[新機能 – Amazon EventBridge で Amazon S3 イベント通知を使用する](https://aws.amazon.com/jp/blogs/news/new-use-amazon-s3-event-notifications-with-amazon-eventbridge/) の話が出て、確認したらCDKで実装できそうだった。
EventBridgeにちゃんと触れたことがなかったので、やってみようと考えた。

# S3バケットの設定

## マネージメントコンソール

以下の画像の通り、イベント通知の下にAmazon EventBridgeの欄があり、これを有効にすればEventBridgeへの通知が可能になる。
![](/images/s3-eventnotification-with-eventbridge/management_console.png =500x)

## CDK

CDKの2系だと、[v2.7.0](https://github.com/aws/aws-cdk/releases/tag/v2.7.0)から対応が入っており、筆者はv2.8.0で実施した。

### class Bucket (construct)

上記のリリースページに`Added new Bucket property to enable this feature (eventBridgeEnabled: true)`とある通り、eventBridgeEnabledを有効にしたBucketを作ればできるはずだった。[class Bucket (construct)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.Bucket.html#eventbridgeenabled)でもそう書いてあるように見えるが、実際にリソースは作成に失敗してしまう。

```
    // S3 バケット作成(こちらだと作成できなかった)
    // const bucket = new Bucket(this, `${this.stackName}-Bucket`, {
    //   eventBridgeEnabled: true
    // })
```

constructsが内部的に作っていると思われるLambdaが以下のようなエラーをCloudWatch Logsに吐き、結果としてスタックで定義したリソースは作成されなかった。

```
4:10:23 PM | CREATE_FAILED        | Custom::S3BucketNotifications | AppStack-Bucket/Notifications
Received response status [FAILED] from custom resource. Message returned: Error: Parameter valida
4:10:23 PM | CREATE_FAILED        | Custom::S3BucketNotifications | AppStackBucketNotifications3C
718F75
Received response status [FAILED] from custom resource. Message returned: Error: Parameter valida
tion failed:
Unknown parameter in NotificationConfiguration: "EventBridgeConfiguration", must be one of: Topic
Configurations, QueueConfigurations, LambdaFunctionConfigurations. See the details in CloudWatch
Log Stream: 2022/01/23/[$LATEST]cfcaa42e5fcc49af90531885959d7e41 (RequestId: d6c1cd22-5e2e-4aa0-b
210-2be5f0535d4a)
```

後日確認したが、[CDK v2.9.0](https://github.com/aws/aws-cdk/releases/tag/v2.9.0)でrevertされていた。

### CfnBucket

[class CfnBucket (construct)のnotificationConfiguration](https://docs.aws.amazon.com/cdk/api/v2//docs/aws-cdk-lib.aws_s3.CfnBucket.html#notificationconfiguration)で[eventBridgeConfiguration](https://docs.aws.amazon.com/cdk/api/v2//docs/aws-cdk-lib.aws_s3.CfnBucket.NotificationConfigurationProperty.html#eventbridgeconfiguration)の設定はできた。

```
    // S3 バケット作成
    const bucket = new CfnBucket(this, `${this.stackName}-Bucket`,{
      notificationConfiguration: {
        eventBridgeConfiguration: {
          eventBridgeEnabled: true
        }
      },
    })
```

# EventBridgeの設定

EventBridgeとEventBridgeによって駆動されるLambdaを作成する。
Lambdaは受け取ったイベントを標準出力(CloudWatch Logsに出力)するだけのものとした。

EventBridgeの挙動を確認するために、イベントパターンにシンプルな設定パターン(パターン1)とリソースの指定やトランスフォームを設定するパターン(パターン2)の2つで試した。

## パターン1

```
    // Lambda Function 作成
    const fn = new Function(this, `${this.stackName}-Function`, {
      functionName: `${this.stackName}-Function`,
      code: Code.fromInline( `
        // https://docs.aws.amazon.com/lambda/latest/dg/nodejs-handler.html
        exports.handler =  async function(event, context) {
          console.log("EVENT: "+ JSON.stringify(event, null, 2))
          return context.logStreamName
        }
      `),
      runtime: Runtime.NODEJS_14_X,
      handler: "index.handler",
      timeout: cdk.Duration.seconds(3),
    })

    // EventBridge Rule作成
    const rule = new Rule(this, `${this.stackName}-Rule`, {
      ruleName: `${this.stackName}-Rule`,
      eventPattern: {
        "source": ["aws.s3"],
        "detailType": ["Object Created"],
      },
      targets: [new LambdaFunction(fn)]
    })
```

上記の設定でアカウント内の**Amazon EventBridgeを有効にした任意のS3バケット**に空ファイルのオブジェクトを作成すると、Lambdaのログで以下のように出力がありました

```
2022-01-26T13:51:52.741Z	74dfbbc9-3a2c-48f8-9037-912f13e3b933	INFO	EVENT: 
{
    "version": "0",
    "id": "6b730fd2-7259-9cbb-c069-09d13fab2931",
    "detail-type": "Object Created",
    "source": "aws.s3",
    "account": "XXXXXXXXXXXX",
    "time": "2022-01-26T13:51:48Z",
    "region": "ap-northeast-1",
    "resources": [
        "arn:aws:s3:::s3-eventbridge-test-bucket"
    ],
    "detail": {
        "version": "0",
        "bucket": {
            "name": "s3-eventbridge-test-bucket"
        },
        "object": {
            "key": "test.txt",
            "size": 0,
            "etag": "d41d8cd98f00b204e9800998ecf8427e",
            "sequencer": "0061F151F4C8285748"
        },
        "request-id": "N21BD34Q1F88R1XC",
        "requester": "XXXXXXXXXXXX",
        "source-ip-address": "XXX.XXX.XXX.XXX",
        "reason": "PutObject"
    }
}
```

[ドキュメントに記載されているイベントの形式](https://docs.aws.amazon.com/AmazonS3/latest/userguide//ev-events.html)と同じ形式で出力されていました。(ただし、ドキュメント上 objectに含まれるversionは、S3のバージョニングを有効にしていなかったためか出力されていなかった。)

このときこのルールに紐付いているCloudWatch Metricsも以下のように出力されていることを確認しました。

![](/images/s3-eventnotification-with-eventbridge/cloudwatch_metrics_pattern.png =500x)


## パターン2

パターン1に以下の設定を加え、パターン2としました

- eventPattern の resources にCDKで作成したS3バケットを指定
- ターゲットにわたすイベントを指定
    - LambdaにわたすイベントをEventBridgeが発行したイベントから必要な項目(ここでは、イベント発生時刻・S3オブジェクトキーとそのサイズのみ)で構成されるオブジェクトに設定

```
    // Lambda Function 作成
    const fn = ・・・・

    // EventBridge 作成
    const rule = new Rule(this, `${this.stackName}-Rule`, {
      ruleName: `${this.stackName}-Rule`,
      eventPattern: {
        "source": ["aws.s3"],
        "detailType": ["Object Created"],
        "resources": [bucket.attrArn]
      },
      targets: [new LambdaFunction(fn, {
        event: RuleTargetInput.fromObject({
          time: EventField.time,
          object: {
            key: EventField.fromPath('$.detail.object.key'),
            size: EventField.fromPath('$.detail.object.size')
          }
        })
      })]
    })
```

上記の設定で**CDKで作成したS3バケット**(変数bucketのS3バケット)にオブジェクトを作成すると、Lambdaのログで以下のように出力がありました。

```
2022-01-26T14:19:14.323Z	9c8e278b-75b8-45f1-a275-0781a3bc0d19	INFO	EVENT: 
{
    "time": "2022-01-26T14:19:10Z",
    "object": {
        "key": "test.txt",
        "size": 0
    }
}
```

上記の出力より、パターン1で出力したイベントのうち、必要な項目のみ指定したオブジェクトをイベントとしてLambdaが受けていることがわかります。
このときの CloudWatch Metricsは、パターン1と同様にTriggeredRulesとInvocationsが1ずつ表示されていました。

ちなみに、試しにイベントパターンに指定していないバケットでオブジェクト作成してみましたが、ルールのトリガーは発生しませんでした。

# リポジトリ

https://github.com/nmemoto/aws-s3-eventbridge-cdk にパターン2の構成をリポジトリを置きました。もし実行される場合は、バケット名を変更しないと実行できないです。

# まとめ

- S3イベント通知のEventBridge有効化設定をバケット毎にできるようになった。これを有効にすることによりバケット内の全てのイベント通知をEventBridgeで受けれるようになる。
- 筆者の環境(CDK v2.8.0)で [Bucket(Constructs)のeventbridgeenabled](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.Bucket.html#eventbridgeenabled)をtrueにしても正常動作しなかった。[CfnBucket(Constructs)のnotificationConfiguration](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.CfnBucket.html#notificationconfiguration) [eventBridgeConfiguration](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.CfnBucket.NotificationConfigurationProperty.html#eventbridgeconfiguration)の設定を使ったらできた。
- EventBridgeのルールのイベントパターンでEventBridgeのルールをトリガーする条件を指定でき、ターゲットで実行したい処理(Lambda等)と実行したい処理に渡すイベントの形式を指定できる。
