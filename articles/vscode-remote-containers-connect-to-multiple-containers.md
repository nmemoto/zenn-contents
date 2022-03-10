---
title: "VSCode Remote Containers で複数コンテナへの接続を試したけどイマイチ"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

# やったこと

[Connect to multiple containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers)を見て、1リポジトリ内にGoとPythonの開発環境用コンテナを起動する構成である[nmemoto/vscode-remote-containers-connect-to-multiple-containers](https://github.com/nmemoto/vscode-remote-containers-connect-to-multiple-containers)を作成してみた。


# Connect to multiple containers のページに書かれていること

## 概要

この仕組みを使うと、1リポジトリ内でVSCode Remote Containersの仕組みで起動するコンテナを複数作成することができる。
2022年2月23日現在、VSCodeの1つのウィンドウ上で同時に複数のコンテナに接続することはできないが、それぞれのウィンドウから1つずつコンテナにアタッチして使用することができる。

## 構成例について

[Connect to multiple containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers) の構成例は、プロジェクトルート直下にdocker-compose.ymlを配置し、docker-compose内で定義したservice毎に.devcontainer.jsonを構成している。
ページの下の方にある[Extending a Docker Compose file when connecting to two containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers#_extending-a-docker-compose-file-when-connecting-to-two-containers)に記載されている構成も、docker-compose.ymlをVSCode Remote Containers 以外でも使用する場合に、Remote Containers 固有の設定をdocker-compose.devcontainer.ymlに記載しているだけで基本的に同じ構成である。

## 利用時の注意点

- VSCode上でソース管理が正しく機能するために、コンテナ内で.gitを含んだプロジェクトルート以下のディレクトリを認識できるようにボリュームマウント設定を入れる必要がある。
- 全てのdevcontainer.jsonには`"shutdownAction":"none"`を入れる必要がある。あるコンテナのVSCodeのウィンドウを閉じたときにdocker-compose.ymlで構成したコンテナ全体が停止するのを防ぐため。

## プロジェクトの開き方

通常のVSCode Remote Containersのプロジェクトであれば、ソースコードをcloneしてプロジェクト直下でVSCodeを開き、ウィンドウ左下のボタン(VSCode 拡張の[Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)を導入してたら出るアイコン)を押せば開発環境が構築できる。

この手順だと、通常の開き方と比べ手数が少し増える点と、増えた手順である`Remote-Containers: Open Folder in Container`はを実行する正直気の進む手順ではなかった。
端末(ターミナル)操作でプロジェクトルート直下まで移動し
さらに.devcontainerディレクトリもしくは.devcontainer.jsonの格納してあるディレクトリにcdして`code .`すれば、通常の手順と同じように

# nmemoto/vscode-remote-containers-connect-to-multiple-containers の説明

ほとんど[Connect to multiple containers](https://code.visualstudio.com/remote/advancedcontainers/connect-multiple-containers)に書いてあることとほとんど同じことをしているが、以下の点を試してみた。

- .devcontainer/Dockerfile を使用してdocker-compose.ymlではbuildオプションを使って記載した
    - 

- プロジェクトルート直下のファイルをConnect to multiple containersの構成では開きにくいので、[multi root workspace](https://code.visualstudio.com/docs/editor/multi-root-workspaces)の設定ファイルを各

# やってみて考えたこと

## この構成のよいところ

- 各開発環境に必要な設定を独立に保持・管理できるところ

## この構成のイマイチなところ

- ウィンドウを複数用意しなければならないところ
- 通常のRemote Containersのプロジェクトと比べて、ソースコードを触るまでの手順が加わる、また今回の構成以外でその手順を行う頻度が低いため、


これまで自分が作ってきたリポジトリはこれだけの手順で必要な環境が作成されるようにしてきたが、今回の構成だとプロジェクトルートでVSCodeを開いてしまうと、ウィンドウを増やす環境上で

自分の場合は、ghqとpecoを使ってソースコード直下まで移動して`code .`コマンドでVSCodeのウィンドウを開いているが、`code .`

プロジェクトルート直下においた
