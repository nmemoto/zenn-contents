---
title: "dev container CLIとDevelopment Containers Specificationについて"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "DevelopmentContainers"]
published: true
---

MicrosoftとGitHubのdevelopment containerチームがDevelopment Containers Specificationの一部としてdev container CLIをオープンソースのコマンドインターフェースとしてリリースしたとのこと、以下の記事で発表されていました。

https://code.visualstudio.com/blogs/2022/05/18/dev-container-cli

オープンソースとしてリリースされたdev container CLIのリポジトリは以下。

https://github.com/devcontainers/cli

Development Containers Specification のリポジトリは以下。

https://github.com/devcontainers/spec

Development Containersの公式ページは以下。

https://devcontainers.github.io/

これらの資料をもとに情報を整理しました。

## Development Containersの役割

「一貫性のある予測可能な環境は、生産的で楽しいソフトウェア開発体験の鍵です。」と[記事](https://code.visualstudio.com/blogs/2022/05/18/dev-container-cli)冒頭に書いてありました。筆者はVSCode拡張の[Remote Containers](https://code.visualstudio.com/docs/remote/containers)にその魅力を感じてこれがリリースされた当初から使用していました。Remote Containersは、これを利用する個人や複数人の開発環境に"一貫性のある予測可能な環境"を提供する仕組みで、dev container CLIはRemote Containersで内部的に使われていた仕組みだと私は理解していました。

ところが、今回[公開された資料](https://devcontainers.github.io/overview#Development-vs-production)の中に「開発コンテナ(developement container)は、デプロイの準備が整う前にアプリケーションを開発するための環境を定義します」とあり、単なる個々人の開発環境のみの話でだけではなく、CIで行われるビルドやテストにもカバーする領域を増やしていくような記載や図が含まれていました。

![個人の開発サイクルでのコンテナ利用・個人より外側(チーム・組織)の開発サイクルでのコンテナ利用・本番コンテナ利用の用途やコードの状態の違い](https://code.visualstudio.com/assets/blogs/2022/05/18/dev-container-stages.png)

上記のようなdevelopment containersの役割を満たせるDevelopment Containers Specificationという仕様を作っていく、またdevelopment containersを作れるのはVSCodeやCodespaceの特定の製品に限ったものでなく一般的なものにしていく、という宣言を今回したという理解を私はしました。

## devcontainer.jsonとdev container CLIについて

[devcontainer.json](https://code.visualstudio.com/docs/remote/devcontainerjson-reference)はこれまでRemote Containersで起動するコンテナを設定するために記述してきました。これからはdevcontainer.jsonは、dev container CLIが起動するコンテナの設定を記述するものとなり、任意のタイミングでdevcontainer.jsonに設定されたコンテナイメージを作成し、起動できるものとなりました。
これにより、CIでも使用できるコンテナをdevcontainer.jsonから作成できるようになったと言えそうです。

筆者はまだ試せていないですが、dev container CLIの使い方は以下にあります。

https://code.visualstudio.com/blogs/2022/05/18/dev-container-cli#_how-can-i-try-it

https://github.com/devcontainers/cli


また、[こちら](https://code.visualstudio.com/blogs/2022/05/18/dev-container-cli#_what-is-the-dev-container-cli)にある通り、dev container CLIはDevelopment Containers Specificationのリファレンス実装という立ち位置でもあります。

dev container CLIがVSCode特有の仕組みでなく一般的なものとするためなのか、VSCode固有のdevcontainer.jsonの設定値は[VS Code specific properties](https://code.visualstudio.com/docs/remote/devcontainerjson-reference#_vs-code-specific-properties)にある通り、`customizations.vscode`として記載されるように変わったようです。

## 感想

Remote Containersのような仕組みが他のIDEにも組み込まれれば、開発者の開発環境の差異がより小さくなり、より開発そのものに集中できる点はかなり喜ばしいことではないでしょうか。個人的な希望ですが、JetBrains製のIDEを使っている人の割合は多いと思うので、JetBrainsの方にDevelopment Container Specificationの仕様策定に一緒に取り組んでほしいです。

また、dev container CLIでdevcontainer.jsonに記述したコンテナを任意のタイミングで起動できるようになり、devcontainer.jsonの活用可能性が広がったこと自体が素晴らしいことだと感じました。早くよいユースケースを見てみたい。