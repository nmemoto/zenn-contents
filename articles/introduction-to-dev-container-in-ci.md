---
title: "VSCode May 2022で紹介されていたDev Container in CIについて調べてみた"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode","githubactions","remotecontainers"]
published: true
---

VSCode May 2022 (version 1.68)にDevelopment Container Specification関連の発表があった。

https://code.visualstudio.com/updates/v1_68#_development-container-specification

その中に、Dev Container in CIという項目があり、それについて調べてみた。

## Dev Container in CI

GitHub ActionとAzure DevOpsでdevcontainer.jsonで定義されたDevelopment Containerを使えるようになったようだ。

Development Containerについては、以下の記事を参照されたし。
https://zenn.dev/nmemoto/articles/devcontainer-cli

以降、本記事ではGitHub ActionでのDevelopment Containerの利用について記載する。

GitHub ActionでDevelopment Container使用に関するドキュメントは以下で確認できる。
https://github.com/devcontainers/ci/blob/main/docs/github-action.md

上記を見ると、`devcontainers/ci@v0.2`というActionsを用いていることがわかる。Actionsについては[GitHub Docs > GitHub Actions > Learn GitHub Actions > Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#actions)を参照のこと。

また、`devcontainers/ci`のソースは以下。
https://github.com/devcontainers/ci

[devcontainers/ciのGitHub Actionのドキュメント](https://github.com/devcontainers/ci/blob/main/docs/github-action.md#getting-started)を参考に、以下のリポジトリで実際にDevelopement ContainerでのCI(と言っても、go testしているだけだが)をやってみた。

https://github.com/nmemoto/try-devcontainer-ci/blob/main/.github/workflows/sample.yaml

基本的にはビルドしたDocker Imageを保存するDocker Registryが必要で今回はこのリポジトリのGitHub Container Registryを使用した。
ただし、これを使用するためにはリポジトリをpublicにしておく必要がある。

```yaml
name: 'build' 
on: # rebuild any PRs and main branch changes
  pull_request:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:

      - name: Checkout (GitHub)
        uses: actions/checkout@v2

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v1 
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and run dev container task
        uses: devcontainers/ci@v0.2
        with:
          imageName: ghcr.io/nmemoto/try-devcontainer-ci
          runCmd: go test
```

リポジトリ内の.devcontainer/devcontainer.jsonは以下のみで、ローカル開発で使用したDev container featuresの設定しかない。

```json
{
	"name": "vscode-golang-template",
	"image": "mcr.microsoft.com/vscode/devcontainers/base:debian",
	"features": {
		"golang": "latest",
		"github-cli": "latest"
	},
	"remoteUser": "vscode"
}
```


`go test`は以下で実行されていた。
https://github.com/nmemoto/try-devcontainer-ci/runs/6828092540?check_suite_focus=true#step:4:2796


## 所感

以前より、Dev container featuresを用いることにより自力でいちいち必要な設定を調べず、Dockerfileに設定を入れ込まなくてもローカル開発を始めることができるようになっていた。
これと同じような感覚で、CIの実行環境として改めて何か準備することなく、CIを始められる選択肢ができたのは良いことのように思う。
これを使えばCIで実行される環境をローカル開発で使用できるので、何度もリポジトリにプッシュしてCIを動かすみたいな試行錯誤を減らすことができるのではないでしょうか。

CIで使う際は、それを念頭においた設定をする必要があり、ローカル開発環境にも多少影響があるのではとも考えた。例えば、今回の設定でDev container featuresで導入したgolangのバージョンにはlatestを指定したが、CI上でバージョンを固定にしないのは好ましくないように思う。
また、明示的にDockerfileに設定内容を記載できていないと、作成したDocker Imageがブラックボックスになりやすいかもしれない。
