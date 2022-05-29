---
title: "dev container CLIとDevelopment Containers Specificationについて"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "DevContainerCLI"]
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

これらの資料をもとにDevelopment Containersとは何か、dev container CLIが何を実現したいかを整理しました。

## Development Containersの役割と今回の発表の意義

「一貫性のある予測可能な環境は、生産的で楽しいソフトウェア開発体験の鍵です。」と[記事](https://code.visualstudio.com/blogs/2022/05/18/dev-container-cli)冒頭に書いてありました。筆者はVSCode拡張の[Remote Containers](https://code.visualstudio.com/docs/remote/containers)にその魅力を感じてこれがリリースされた当初から使用していました。Remote Containersがこれを利用する開発者の開発環境に"一貫性のある予測可能な環境"を提供する仕組みであり、基本的にはdev container CLIはRemote Containersで内部的に使われていた仕組みだと、筆者はこれまで理解していました。(実際には[dev container CLIがドキュメント上登場したタイミング](https://github.com/microsoft/vscode-docs/commit/f756b40098b1e8945ca2e0e830fd1d6d99f6b562)からRemote Containersとは独立にコマンド実行できるようにはなっていたようです。)

ところが、今回の発表により、dev container CLIはオープンソースとして公開されRemote Containersのみで使用するツールではなくなりました。また、dev container CLIによって作られるDevelepment Containersは各々の開発者のみで使うものでもなくなりました。

今回[公開された資料](https://devcontainers.github.io/overview#Development-vs-production)の中に「開発コンテナ(Developement Container)は、デプロイの準備が整う前にアプリケーションを開発するための環境を定義します」とあり、各々の開発者の開発環境のみの話でだけではなく、CIで行われるビルドやテストにもカバーする領域を増やしていくような記載や図が含まれていました。

![開発者個人の開発サイクルでのコンテナ利用・CI(ビルドやテストの開発サイクル)の中でのコンテナ利用・本番コンテナ利用の用途やコードの状態の違い](https://code.visualstudio.com/assets/blogs/2022/05/18/dev-container-stages.png)


上記のようなDevelopment Containersの役割を満たすDevelopment Containers Specificationというオープンな仕様を作っていく、またdev container CLIをオープンソースとして公開することでDevelopment Containersを作れるのはVSCodeやCodespaceという特定の製品に限ったものとしない、他のツールでも使えるものとしていく、という宣言を今回したと筆者は理解しました。

## devcontainer.jsonとdev container CLIについて

[devcontainer.json](https://code.visualstudio.com/docs/remote/devcontainerjson-reference)は基本的にはRemote Containersで起動するコンテナを設定するために記述してきました。これからはdevcontainer.jsonは、dev container CLIが起動するコンテナの設定を記述するものとなり、任意のタイミングでdevcontainer.jsonに設定されたコンテナイメージを作成・そのイメージからコンテナ起動できるものとなりました。
これにより、CIでも使用できるコンテナをdevcontainer.jsonから作成できるようになったと言えそうです。

筆者はまだ試せていないですが、dev container CLIの使い方は以下にあります。

https://code.visualstudio.com/blogs/2022/05/18/dev-container-cli#_how-can-i-try-it
https://github.com/devcontainers/cli

また、[こちら](https://code.visualstudio.com/blogs/2022/05/18/dev-container-cli#_what-is-the-dev-container-cli)にある通り、dev container CLIはDevelopment Containers Specificationのリファレンス実装という立ち位置でもあります。

dev container CLIがVSCode特有の仕組みでなく一般的なものとするためなのか、VSCode固有のdevcontainer.jsonの設定値は[VS Code specific properties](https://code.visualstudio.com/docs/remote/devcontainerjson-reference#_vs-code-specific-properties)にある通り、`customizations.vscode`として記載されるように変わったようです。

## 感想

Remote Containersのような仕組みが他のIDEにも組み込まれれば、開発者間の開発環境の差異がより小さくなり、より開発そのものに集中できる点はかなり喜ばしいことではないでしょうか。JetBrains製のIDEを使っている人の割合は多いと思うので、JetBrainsの方もDevelopment Container Specificationの仕様策定に一緒に取り組んで頂き、同じリポジトリでの開発者共通の開発環境を実現して頂けると個人的にはとても嬉しいです。

また、dev container CLIでdevcontainer.jsonに記述したコンテナを任意のタイミングで起動できるようになり、devcontainer.jsonの活用可能性が広がったこと自体が素晴らしいことだと感じました。完全に推測ですが、GitHub Actionsといい感じに統合され、CI環境でのDevelopment Containersの利用が楽にできるようになるのでは思ったりします。
よいユースケースが今後出てくるのが楽しみです。

## 2022/5/30 追記

dev container CLIは[その存在が示されたとき](https://github.com/microsoft/vscode-docs/commit/f756b40098b1e8945ca2e0e830fd1d6d99f6b562)からRemote Containersとは独立に実行できるコマンドではあったようです。
ただ、当時dev container CLIのソースは発表されておらず、またその使い方がVSCode(Remote Containers)のドキュメント上のみで示されていたため、dev container CLIは事実上Remote Containersのためのツールと言っても過言ではなかったのではないかと思っています。
