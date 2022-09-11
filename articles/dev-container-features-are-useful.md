---
title: "1リポジトリで2つ以上の開発言語のローカル開発環境をDevcontainerで作るとき、Dev Container Featuresが便利"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "remotecontainers", "devcontainer"]
published: true
---

# やったこと

2つの開発言語(GoとPython)のソースコードを1リポジトリで管理するときのローカル開発環境を Devcontainer(VSCode Remote  - Containers) でどのように整備するか検討した。

調査の結果、下記の3つの案が取りうる構成だとわかった

- [Connect to multiple containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers)にあるように、開発言語毎に任意の設定のコンテナを作成する
- devcontainer.jsonで指定するDockerfile内に複数言語の任意の設定を入れ、一つのコンテナを作成する
- [Dev Container Features (preview)](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)で予め定義された設定を導入し、一つのコンテナを作成する

最後の[Dev Container Features (preview)](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)を使用する案を採用し、[サンプルリポジトリ](https://github.com/nmemoto/vscode-remote-containers-go-and-python)を作るに至った。
https://github.com/nmemoto/vscode-remote-containers-go-and-python

理由は、以下3点である。
- 今回は、開発環境のために細かい設定をする必要がなくDev Container Features (preview)で提供されている[golang](https://github.com/devcontainers/features/tree/main/src/go)と[python](https://github.com/devcontainers/features/tree/main/src/python)をそのまま使用して問題なさそうだった
- Dev Container Features (preview)と書いてあるとおり正式には提供されている機能でないが、仮にこの機能が廃止もしくは変更されたとしても、2案目に挙げたような代替のDockerfileを定義することも時間をかければ可能と考え、採用するデメリットはあまりないと考えた
- 1案目のConnect to multiple containersの案は、個々のコンテナ毎にVSCodeのウィンドウを起動させて使用する必要がある、プロジェクトルート直下のファイルを編集しづらいなど、実際利用する際に不便なことが多かった

# 調査したこと/やったことの詳細

## Dev Container Features (preview) 

[Dev Container Features (preview)](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)は、 [Development Container Specification](https://containers.dev/implementors/features/)で規定されてる提案中の仕様で、以下でFeaturesはインストールコードと開発コンテナ設定の自己充足的で共有可能な単位であると紹介されている。
https://github.com/devcontainers/features

実際、Dockerfileに`apt-get install ....`や`curl ...`等開発環境のためのセットアップを書かなくても任意のツール導入ができる点やVSCode上で使いやすい設定が入った上で使用開始できる点が優れていると筆者も感じている。

最も基本的な使い方は以下のように`devcontainer.json`内に導入したいツール設定する。そして、プロジェクトをコンテナで起動すると、Dockerビルド時に指定したツールがコンテナ内に導入される。

```json
"features": {
    "ghcr.io/devcontainers/features/github-cli:1": {
        "version": "latest"
    }
}
```

1つのコンテナ内に多数の言語の開発環境を揃える場合は、以下のように設定すれば導入できる。

```json
  "features": {
    "ghcr.io/devcontainers/features/python:1": {
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/go:1": {
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/ruby:1": {
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/java:1": {
        "version": "latest"
    },
    "ghcr.io/devcontainers/features/rust:1": {
        "version": "latest"
    }
  }
```

ただし、書いてあるとおりpreview版の仕様・機能であることと、上記のような設定で導入できるツールや環境は限られるため、その点確認の上使用する必要がある。
導入可能なツールは https://github.com/devcontainers/features/tree/main/src 内のディレクトリ毎のREADMEに記載されている。
Dev Container Featuresを用いた任意のツール導入処理を[devcontainers/feature-template](https://github.com/devcontainers/feature-template)を使って自作することもできる。
このテンプレートを使用し、新しいFeaturesの仕様に従えば、GitHub Container Registryを使用して気軽に自作のDev Container Featuresを公開することができる。

## サンプルリポジトリの設定説明

GoとPythonの開発環境のためのDev Container Featuresを使用した[サンプルリポジトリ](https://github.com/nmemoto/vscode-remote-containers-go-and-python)を作った。
設定は[.devcontainer.json](https://github.com/nmemoto/vscode-remote-containers-go-and-python/blob/main/.devcontainer.json)のみである。

```json:.devcontainer.json
{
  "name": "go-and-python",
  "image": "mcr.microsoft.com/devcontainers/base:debian",
  "shutdownAction": "stopContainer",
  "remoteUser": "vscode",
  "features": {
    "ghcr.io/devcontainers/features/python:1": {
      "version": "latest",
      "installJupyterlab": true
    },
    "ghcr.io/devcontainers/features/go:1": {
      "version": "latest"
    }
  },
  "postStartCommand": "pip install -r python-workspace/requirements.txt"
}
```

"features"では今回の開発環境で用意したい[go](https://github.com/devcontainers/features/tree/main/src/go)と[python](https://github.com/devcontainers/features/tree/main/src/python)を指定した。
featuresに関するgoの設定は[こちら](https://github.com/devcontainers/features/blob/main/src/go/devcontainer-feature.json)、pythonの設定は[こちら](https://github.com/devcontainers/features/blob/main/src/python/devcontainer-feature.json)にある。それぞれ確認するとcustomizations.vscode.extensions/customizations.vscode.settings/containerEnv等の設定がされており、開発に適した設定が予めされていることがわかる。これにより、起動してすぐにデバッグ可能な環境を用意することができる。

また、FeatureのPythonにはinstallJupyterlabというオプションがあり、これを利用してJupyterlabを導入した。
Feature毎に使用できるオプションが変わるので、各々のFeaturesのREADMEを参照されたし。(以下はpythonのREADME)
https://github.com/devcontainers/features/blob/main/src/python/README.md#options

もともと2つの開発言語(GoとPython)のソースコードを1リポジトリで管理するときの開発環境として Dev Container Featuresを検討していたが、1つの開発言語の環境でも楽に用意できそうなことがわかる。(なので、過去タイトルが1リポジトリで**1つ以上**の開発言語の〜としていた。)
ただし、1つの開発言語しか使用しない場合は、Featuresをその言語のコンテナイメージを[devcontainers/images](https://github.com/devcontainers/images)から探して使用するのが良いと考えるにいたり、タイトルを更新することにした。その理由と、回避策については以下の記事にまとめているので、興味のある方は参照されたし。
https://zenn.dev/nmemoto/articles/buildcachefrom-is-useful-in-devcontainer

"image"では[mcr.microsoft.com/devcontainers/base:debian](https://github.com/devcontainers/images/tree/main/src/base-debian)を指定した。Development Containerで使用するコンテナイメージを管理する[devcontainers/images](https://github.com/devcontainers/images)で提供している一番シンプルなDockerイメージを使用した。

"postStartCommand"にはコンテナ起動時に実行される処理を記載しており、ここではPythonのスクリプト内でimportしているライブラリを予めインストールする処理を記載している。

## Connect to multiple containers で紹介されている構成は使い勝手がよくなかった

冒頭で第1案として載せた[Connect to multiple containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers)で紹介されている構成は、1リポジトリに複数コンテナでの開発環境を用意する必要があるときに使う構成例である。
https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers

冒頭に記載されているが、現在はVSCodeのウィンドウ一つにつき1コンテナのみ接続が可能となっているため、複数コンテナに接続して開発する際は複数のウィンドウを起動することが必須となる。

一つ一つのコンテナで必要となる開発環境の構築手順が非常に複雑な場合は、この構成を取ることによりそれらを各コンテナで明確に分けることができその点はメリットとなりうる。
しかしそうでない場合、この構成をとるメリットはほぼないと考えられる。
まず、単純に他言語の開発環境に切り替えるときウィンドウの行き来するのが地味に面倒である。
また、上記の構成例の場合、コンテナでプロジェクトを開いた際、左ペインのExplorerにプロジェクトルート直下のファイルは表示されないのも少し気になる点であった。(ただし、プロジェクトルートディレクトリ以下でボリュームマウントはされるため、ターミナル上でのファイルアクセスは可能である。そのため、コンテナ内のルートディレクトリで`code ../README.md`すればファイル編集はコンテナ内でも行える。これもただ面倒である。)
[Multi-root Workspaces](https://code.visualstudio.com/docs/editor/multi-root-workspaces)を使えば、Explorerに表示できないこともないが、そこまでして設定を増やしたくもない。
さらに、Remote Containersでプロジェクトを開くとき、ディレクトリを選ぶ作業やそれを複数回するという手順を行う必要があり、若干とは言え1コンテナのときと異なる手順となることにも抵抗はあった。

というわけでこの構成を実際に試すところまでやってはみたものの、日の目を見なかったものとしてここで供養する。
https://github.com/nmemoto/vscode-remote-containers-connect-to-multiple-containers

# まとめ

導入できる言語環境やツールは限られるが、1リポジトリで2つ以上の開発言語の開発環境をDevcontainer(VSCode Remote - Containers)で作るときの構成で、Dev Container Features (preview) を使うのが便利だとわかった。
ただし、preview版である点や導入できるツールの種類とその環境は限定されている点に注意する必要がある。
