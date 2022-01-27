---
title: "Zenn 記事の執筆環境を VSCode Remote Containers を使って構築する"
emoji: "💨"
type: "tech"
topics: ["vscode"]
published: true
---

# やりたいこと

GitHub上で管理する Zenn 記事の執筆環境を VSCode Remote Containers で用意したい

# 実際にやったこと

## ベースイメージ

[github.com/microsoft/vscode-remote-try-node](https://github.com/microsoft/vscode-remote-try-node)で使用している [mcr.microsoft.com/vscode/devcontainers/javascript-node](https://github.com/microsoft/vscode-dev-containers/tree/main/containers/javascript-node):0-16-bullseye を使用することとした。

その際、以下の2点を考慮した。
- Zenn CLI は Node.js で動作する
- Zenn 記事を GitHub で管理する
  - Git とコミットの Push 等の Git リモートリポジトリの操作ための SSH クライアントが必要

このイメージはmicrosoft管理のイメージで、Remote Containersを使うために必要十分な設定がされている。
採用したイメージは、nodeをベースイメージとし、[Dockerビルドの処理](https://github.com/microsoft/vscode-dev-containers/blob/v0.215.1/containers/javascript-node/.devcontainer/base.Dockerfile)内で[実行されているシェルスクリプト](https://github.com/microsoft/vscode-dev-containers/blob/v0.215.1/containers/javascript-node/.devcontainer/base.Dockerfile#L23)により開発で最低限必要なgitやopenssh-client等のパッケージが導入された状態で利用を開始できるのが良い。

## Remote Containers 設定

以下のように設定した。

```json:.devcontainer/devcontainer.json
{
  "name": "zenn-contents",
  "image": "mcr.microsoft.com/vscode/devcontainers/javascript-node:0-16-bullseye",
  "extensions": ["GitHub.vscode-pull-request-github"],
  "portsAttributes": {
    "8000": {
      "label": "Zenn Preview",
      "onAutoForward": "openBrowser"
    }
  },
  "features": {
    "github-cli": "latest"
  },
  "postCreateCommand": "npm install",
  "remoteUser": "node"
}
```

[`postsAttributes`](https://code.visualstudio.com/docs/remote/devcontainerjson-reference#_port-attributes)はコンテナ内で起動するプレビューアプリケーションをホスト側で閲覧するためのデフォルトのポートフォワード設定で、`npx zenn preview`のデフォルトの起動ポートの8000を指定した。
`"onAutoForward": "openBrowser"`は、ポートフォワードされたタイミングでこのポートを使うアプリケーションをシステムのデフォルトブラウザで開くために設定した。

`features`は、2022年1月27日現在、Preview 機能である[Dev container features](https://code.visualstudio.com/docs/remote/containers#_dev-container-features-preview)のことを指しており、https://github.com/microsoft/vscode-dev-containers/tree/main/script-library/docs に記載のあるツールについてはこの記法で該当ツールを導入できるようになる。
ここでは`github-cli`を導入している。

`postStartCommand`ではコンテナ起動後に実行されるコマンドを設定できる。
Zennのコンテンツを管理しているこのリポジトリでは、DependabotとGitHub Actionsが勝手にpackage.jsonをアップグレードしてマージしてくれるようにしている。
新しい記事を書くタイミングで最新のmainブランチからブランチを作成すれば、その次にコンテナ起動したタイミングで`postStartCommand`で指定した`npm install`が動作し、自然と最新のzenn-cliを使用できるようになる。
なお、DependabotとGitHub Actionsによる自動マージについては[GitHub ActionsでのDependabotの自動化](https://docs.github.com/ja/code-security/supply-chain-security/keeping-your-dependencies-updated-automatically/automating-dependabot-with-github-actions#common-dependabot-automations)に詳しく記載されていた。

devcontainer.jsonのそれぞれの設定の詳細については、[devcontainer.jsonのリファレンス](https://code.visualstudio.com/docs/remote/devcontainerjson-reference)に記載がある。


# まとめ

- ベースイメージは、[mcr.microsoft.com/vscode/devcontainers/javascript-node](https://github.com/microsoft/vscode-dev-containers/tree/main/containers/javascript-node):0-16-bullseye を使うとよさげ。
- `.devcontainer/devcontainer.json`の`postsAttributes`を使用し、ポートフォワード時にブラウザで`http://localhost:8000`を開くようにする。
- DependabotとGitHub Actionsにpackage.jsonの最新化の処理をさせ、マージさせる。postStartCommand`を使用し、コンテナ起動時に毎回`npm install`を実行させると、新しい記事を書くタイミングで最新のzenn-cliを使用することができる。
