---
title: "1リポジトリで1つ以上の開発言語の開発環境をRemote Containersで作るとき、Dev container featuresが便利"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode"]
published: true
---

# やったこと

2つの開発言語(GoとPython)のソースコードを1リポジトリで管理するときの開発環境を VSCode Remote Containers でどのように整えるか検討した。

調査の結果、下記の3つの案が取りうる構成だとわかった

- [Connect to multiple containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers)にあるように、開発言語毎に任意の設定のコンテナを作成する
- devcontainer.jsonで指定するDockerfile内に複数言語の任意の設定を入れ、一つのコンテナを作成する
- [Dev container features (preview)](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)で予め定義された設定を導入し、一つのコンテナを作成する

以下3点より、最後の[Dev container features (preview)](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)を使用する案を採用し、[サンプルリポジトリ](https://github.com/nmemoto/vscode-remote-containers-go-and-python)を作るに至った。

- 今回は、開発環境のために細かい設定をする必要がなくDev Container features (preview)で提供されている[golang](https://github.com/microsoft/vscode-dev-containers/blob/bc459941115141bf51239398aea0ef833d7989ee/script-library/container-features/src/devcontainer-features.json#L671-L721)と[python](https://github.com/microsoft/vscode-dev-containers/blob/bc459941115141bf51239398aea0ef833d7989ee/script-library/container-features/src/devcontainer-features.json#L601-L670)の設定をそのまま使用して問題なさそうだった
- Dev container features (preview)と書いてあるとおり正式には提供されている機能でないが、仮にこの機能が廃止もしくは変更されたとしても、2案目に挙げたような代替のDockerfileを定義することも時間をかければ可能と考え、採用するデメリットはあまりないと考えた
- 1案目のConnect to multiple containersの案は、個々のコンテナ毎にVSCodeのウィンドウを起動させて使用する必要がある、プロジェクトルート直下のファイルを編集しづらいなど、実際利用する際に不便なことが多かった

# 調査したこと/やったことの詳細

## Dev container features (preview) 

[Dev container features (preview)](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)は、 Remote Containersで提供されている機能で、コンテナ定義をカスタマイズするスムーズな方法と紹介されている。
https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview
実際、Dockerfileに`apt-get install ....`や`curl ...`等開発環境のためのセットアップを書かなくても任意のツール導入ができる点やVSCode上で使いやすい設定が入った上で使用開始できる点が特に優れていると筆者も感じている。

最も基本的な使い方は以下のように`devcontainer.json`内に導入したいツール設定する。そして、プロジェクトをコンテナで起動すると、Dockerビルド時に指定したツールがコンテナ内に導入される。

```json
"features": {
    "github-cli": "latest"
}
```

ただし、上記のような設定で導入できるツールや環境は限られるため、その点確認の上使用する必要がある。
導入可能なツールとその環境のついては[ここ](https://github.com/microsoft/vscode-dev-containers/tree/main/script-library/docs)にあるマークダウンに記載されている。

詳細は、別途[スクラップ](https://zenn.dev/nmemoto/scraps/9eee0f54dc30ed)に調べたことをまとめているので、気になる方は参照されたし。
https://zenn.dev/nmemoto/scraps/9eee0f54dc30ed

## サンプルリポジトリの設定説明

GoとPythonの開発環境のためのDev container featuresを使用した[サンプルリポジトリ](https://github.com/nmemoto/vscode-remote-containers-go-and-python)を作った。

Remote Containersに関わる設定である[.devcontainer.jsonの中身](https://github.com/nmemoto/vscode-remote-containers-go-and-python/blob/2d9c670c8dc5c22ab3b97313c9693d289fa9d8c4/.devcontainer.json)は以下のみである。

```json
{
  "name": "go-and-python",
  "image": "mcr.microsoft.com/vscode/devcontainers/base:debian-11",
  "shutdownAction": "stopContainer",
  "remoteUser": "vscode",
  "features": {
      "python": "latest",
      "golang": "latest"
  },
  "postStartCommand": "pip install -r python-workspace/requirements.txt"
}
```

"image"ではとりあえず[mcr.microsoft.com/vscode/devcontainers/base:debian-11](https://github.com/microsoft/vscode-dev-containers/tree/main/containers/debian)を指定した。[microsoft/vscode-dev-containers](https://github.com/microsoft/vscode-dev-containers)で提供している一番シンプルそうなDockerイメージであったため使用した。

"features"では今回の開発環境で用意したい[golang](https://github.com/microsoft/vscode-dev-containers/blob/main/script-library/docs/go.md)と[python](https://github.com/microsoft/vscode-dev-containers/blob/main/script-library/docs/python.md)を指定した。
golangの設定は[こちら](https://github.com/microsoft/vscode-dev-containers/blob/bc459941115141bf51239398aea0ef833d7989ee/script-library/container-features/src/devcontainer-features.json#L671-L721)、pythonの設定は[こちら](https://github.com/microsoft/vscode-dev-containers/blob/bc459941115141bf51239398aea0ef833d7989ee/script-library/container-features/src/devcontainer-features.json#L601-L670)にある。それぞれ確認するとextensions/containerEnv/settings等の設定がされており、開発に適した設定が予めされていることがわかる。これにより、起動してすぐにデバッグ可能な環境を用意することができる。

本記事では、1リポジトリで1つ以上の開発言語の開発環境をフォーカスしたものになっているが、Dev container featuresを使えば、単一の言語環境でも楽に開発可能な環境素早く用意できそうなことがこれからわかる。

"postStartCommand"にはコンテナ起動時に実行される処理を記載しており、ここではPythonのスクリプト内でimportしているライブラリを予めインストールする処理を記載している。

## Connect to multiple containers で紹介されている構成は使い勝手がよくなかった

冒頭で第1案として載せた[Connect to multiple containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers)で紹介されている構成は、1リポジトリに複数コンテナでの開発環境を用意する必要があるときに使う構成例である。
https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers

冒頭に記載されているが、現在はVSCodeのウィンドウ一つにつき1コンテナのみ接続が可能となっているため、複数コンテナに接続して開発する際は複数のウィンドウを起動することが必須となる。

一つ一つのコンテナで必要となる開発環境の構築手順が非常に複雑な場合は、この構成を取ることによりそれらを書くコンテナで明確に分けることができその点はメリットとなりうる。

しかし、そうでない場合のメリットはほぼないと考えられ、単純に他言語の開発環境に切り替えるときウィンドウの行き来するのが面倒になる。
また、上記の構成例の場合、左ペインのExplorerにプロジェクトルート直下のファイルは表示されないのも少し気になる点であった。(ただし、プロジェクトルートディレクトリ以下でマウントはされるため、ターミナル上でのファイルアクセスは可能である。そのため、コンテナ内のルートディレクトリで`code ../README.md`すればファイル編集はコンテナ内でも行える。)
[Multi-root Workspaces](https://code.visualstudio.com/docs/editor/multi-root-workspaces)を使えば、Explorerに表示できないこともないが、そこまで設定を増やしたくもない。

また、これも個人的な感覚ではあるが、Remote Containersでプロジェクトを開くとき、ディレクトリを選ぶ作業やそれを複数回するという手順をしないといけないことにも抵抗はあった。

というわけでこの構成を実際に試すところまでやってはみたものの、日の目を見なかったものとしてここで供養する。
https://github.com/nmemoto/vscode-remote-containers-connect-to-multiple-containers

# まとめ

1リポジトリで1つ以上の開発言語の開発環境をRemote Containersで作るときの構成で、Dev container features (preview) を使うのが有用だと考える。
ただし、previewである点、Dev container featuresで導入できるツールや環境は限定されている点、また導入するツールに関して特別な設定を多くするときは自分でDockerfileを書いたほうがよいかもしれない。
