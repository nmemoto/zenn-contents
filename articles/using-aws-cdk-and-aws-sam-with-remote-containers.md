---
title: "VSCode Remote Containers で AWS CDK と AWS SAM を使いコンテナ内部でLambdaを実行する"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "AWS", "CDK", "SAM"]
published: true
---

:::message
本記事の内容は、Intel Macで動作するDocker for MacをDockerホストとしたマシン上でしか検証できていません。
:::

# 必要な設定

ローカルのコンテナ内で`sam local invoke`コマンドを実行することを目的に、VSCode Remote Containers を使ってAWS CDK と AWS SAM を実行できる環境を構築した。
設定は以下の2ファイルです。

```json:.devcontainer/devcontainer.json
{
  "name": "cdk-and-sam",
  "image": "mcr.microsoft.com/vscode/devcontainers/base:debian-11",
  "workspaceMount": "source=${localWorkspaceFolder},target=${localWorkspaceFolder},type=bind",
  "extensions": [
    "amazonwebservices.aws-toolkit-vscode",
  ],
  "features": {
    "node": {
      "version": "lts"
    },
    "docker-from-docker": {
      "version": "latest"
    }
  },
  "remoteUser": "vscode",
  "workspaceFolder": "${localWorkspaceFolder}",
  "postCreateCommand": "bash ./.devcontainer/postCreateCommand.sh"
}
```

```bash:.devcontainer/postCreateCommand.sh
#!/bin/bash
set -ex

# install cdk
# https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/getting_started.html#getting_started_install
npm install -g aws-cdk

# install sam
# https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/serverless-sam-cli-install-linux.html#serverless-sam-cli-install-linux-sam-cli
curl -o /tmp/aws-sam-cli-linux-x86_64.zip -L https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip
cd /tmp
unzip aws-sam-cli-linux-x86_64.zip -d sam-installation
sudo ./sam-installation/install
rm -rf aws-sam-cli-linux-x86_64.zip sam-installation
cd -
```

この設定が入ったAWS CDKのサンプルリポジトリは以下にある。
https://github.com/nmemoto/vscode-remote-containers-cdk-and-sam

なぜ上記の設定でコンテナ内で`sam local invoke`コマンドを実行できるようになるかについて、特に必要だと思われる箇所について以下で説明する。

# 詳細説明

## Dev container features

以下の設定は、Dev container features の機能を用いたものである。
```json
    "features": {
      "node": {
        "version": "lts"
      },
      "docker-from-docker": {
        "version": "latest"
      }
    },
```

これにより、AWS CDKコマンドの実行環境のためのNode.jsと、AWS CDKやAWS SAM CLIが内部的に使用しているDockerコマンド実行のための環境構築を行っている。
ちなみに、docker-from-docker導入時の処理は[こちら](https://github.com/microsoft/vscode-dev-containers/blob/6ae40d55b753e0af7f23f3da53efd587eecbd5f5/script-library/docker-debian.sh)にあるが、いい感じにホストのDocker Engineを使用できるようにソケットファイルの設定やユーザーの権限設定をやってくれ、面倒な設定を肩代わりしてくれることがわかる。

Dev container features については公式情報だと以下に説明がある。
https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview

また、Dev container featuresについては、筆者も過去に以下でまとめている。
https://zenn.dev/nmemoto/articles/dev-container-features-are-useful#dev-container-features-(preview)

## Workspace Volume Mount

:::message
ここからの説明で使用しているホストという言葉は、Docker Engineを動かしているDockerホストを意味している。
:::

以下の設定により、ホスト側の絶対パスと同じ絶対パスをコンテナ内にも作成、そのパスをコンテナ内のWorkspaceとして登録している。

```json
  "workspaceMount": "source=${localWorkspaceFolder},target=${localWorkspaceFolder},type=bind",
  "workspaceFolder": "${localWorkspaceFolder}",
```

上記で使用している設定や変数については以下に説明がある。
https://code.visualstudio.com/docs/remote/devcontainerjson-reference

ではなぜこの設定が必要なのか、この設定をせずに失敗する事象から説明する。

以下のドキュメントでは、AWS CDK アプリケーションでのLambda関数のローカル実行について記載されている。
https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/serverless-cdk-getting-started.html#serverless-cdk-tutorial-hello-init.title

そこで記載されているコマンドを今回のリポジトリに置き換えると以下となる。

```bash
cdk synth --no-staging
sam local invoke AppStack-Function --no-event -t ./cdk.out/AppStack.template.json
```

Workspace Volume Mountの設定なしだと、コンテナ内のデフォルトのworkspaceFolderは`/workspace/vscode-remote-containers-cdk-and-sam/`となる。
そこからの相対パスの`./app`にCDKプロジェクトが格納されているが、そこで`cdk synth --no-staging`を実行すると、`./cdk.out`以下に`AppStack.template.json`が出力される。このファイルのAppStack-Functionに関係する出力は以下となった。

```json:/workspace/vscode-remote-containers-cdk-and-sam/app/cdk.out/AppStack.template.json
・・・
    "AppStackFunctionA0C4729X": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
        ・・・・・・・
        "FunctionName": "AppStack-Function",
        "Handler": "index.handler",
        "Runtime": "nodejs14.x",
        "Timeout": 3
      },
      "DependsOn": [
        "AppStackFunctionServiceRoleB5E28B8A"
      ],
      "Metadata": {
        "aws:cdk:path": "AppStack/AppStack-Function/Resource",
        "aws:asset:path": "/workspace/vscode-remote-containers-cdk-and-sam/app/lambda/my_function",
        "aws:asset:is-bundled": false,
        "aws:asset:property": "Code"
      }
・・・
```

また`sam local invoke AppStack-Function --no-event -t ./cdk.out/AppStack.template.json`の出力は以下となった。
```
・・・・・
docker.errors.APIError: 500 Server Error: Internal Server Error ("b'Mounts denied: \nThe path /workspaces/vscode-remote-containers-cdk-and-sam/app/lambda/my_function is not shared from the host and is not known to Docker.\nYou can configure shared paths from Docker -> Preferences... -> Resources -> File Sharing.\nSee https://docs.docker.com/desktop/mac for more info.'")
[1310] Failed to execute script __main__
```

これは、`AppStack.template.json`内の`AppStack-Function`という名前の関数の"Metadata.aws:asset:path"にあるパスをsam local invoke コマンドがlambda関数のファイルを見つけられず発生している。
言い換えると、コンテナ内に存在する`/workspaces/vscode-remote-containers-cdk-and-sam/app/lambda/my_function`というディレクトリが、sam local invokeコマンドでコンテナを立てるホストのDocker Engineから見ると存在しない(つまり、ホストのディレクトリに存在しない)ためにこの事象が起こっている。

つまり、コンテナ内でsam local invokeコマンド実行時に、ホストのDocker Engineから見て存在するパスにLambda関数のファイルが存在しなくてはならない。
これは、ホストのソースコードを配置しているディレクトリとコンテナ内のソースコードを配置しているディレクトリを同じ構成にしてしまえば実現でき、この設定をWorkspace Volume Mount の設定で行っている。


## sam local invoke 実行時に必要なオプション

当初は上記だけでうまくいくと思っていたが、実際にはsam local invokeコマンドで`--container-host`オプションが必要で、筆者のDocker for Mac上のコンテナの場合、以下のコマンドで実行する必要があった。

```bash
sam local invoke AppStack-Function --no-event -t ./cdk.out/AppStack.template.json --container-host host.docker.internal
```

以下のsam local invokeコマンドのオプション説明に、上記に関して記載があった。
https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-local-invoke.html

これを指定しないと以下のように`No response from invoke container for ~`という出力が出てしまう。

```bash
Invoking index.handler (nodejs14.x)
Skip pulling image and use local one: public.ecr.aws/sam/emulation-nodejs14.x:rapid-1.40.1-x86_64.

Mounting /Users/nmemoto/ghq/github.com/nmemoto/vscode-remote-containers-cdk-and-sam/app/lambda/my_function as /var/task:ro,delegated inside runtime container
No response from invoke container for AppStackFunctionA0C4729X
```

# まとめ

冒頭に挙げた必要な設定を施した、以下のリポジトリをRemote Containersで開く。
https://github.com/nmemoto/vscode-remote-containers-cdk-and-sam

(筆者のDocker for Mac環境では、)CDKプロジェクトのあるディレクトリで、以下コマンド実行することでコンテナ内でLambdaのテスト実行ができる。

```bash
$ cdk synth --no-staging

$ sam local invoke AppStack-Function --no-event -t ./cdk.out/AppStack.template.json --container-host host.docker.internal
Invoking index.handler (nodejs14.x)
Skip pulling image and use local one: public.ecr.aws/sam/emulation-nodejs14.x:rapid-1.40.1-x86_64.

Mounting /Users/nmemoto/ghq/github.com/nmemoto/vscode-remote-containers-cdk-and-sam/app/lambda/my_function as /var/task:ro,delegated inside runtime container
END RequestId: cf333e12-fbc6-4e05-8935-07a8fa5bc0b7
REPORT RequestId: cf333e12-fbc6-4e05-8935-07a8fa5bc0b7  Init Duration: 0.27 ms  Duration: 284.44 ms     Billed Duration: 285 ms Memory Size: 128 MB        Max Memory Used: 128 MB
"Hello from SAM and the CDK!"
```