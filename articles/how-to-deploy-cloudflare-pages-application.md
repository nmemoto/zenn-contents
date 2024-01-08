---
title: "複数のCloudflare Pagesのアプリケーションのソースを管理する単一のGitHubリポジトリからデプロイする"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "cloudflarepages", "githubactions"]
published: true
---

## tl;dr

- Cloudflare Pagesのデプロイ方法は、([公式の分類](https://developers.cloudflare.com/pages/get-started/guide)とは異なるが、)[Git連携](https://developers.cloudflare.com/pages/get-started/guide/#connect-your-git-provider-to-pages)・[Wrangler](https://developers.cloudflare.com/pages/get-started/direct-upload/#wrangler-cli)・[Drag and Drop](https://developers.cloudflare.com/pages/get-started/direct-upload/#drag-and-drop) の3つがある。
- 1リポジトリで単一Cloudflare Pagesアプリケーションを扱う場合は、ビルド設定も一緒に設定できるGit連携するのが一番楽。
- 1リポジトリで複数Cloudflare Pagesアプリケーションを扱う場合は、1アプリケーションまでならGit連携を使用できるが、2つ目以降のアプリケーションに対してGit連携はできないため、その他の2案を検討しなければならず、本番で自動デプロイ運用するならWranglerを使う案の一択となる。
- GitHub Actionsを使う場合、[Wrangler GitHub Action](https://github.com/cloudflare/wrangler-action)を使うとよい。

## やりたいこと

新たに Astro のプロジェクトを作成し、 https://github.com/nmemoto/nmemoto.dev/tree/main/playground で管理するようにしたい。(ちなみに、https://playground.nmemoto.dev で閲覧可能にしたい。)

## 作業前の状態

Cloudflare Pagesで https://nmemoto.dev とそのコンテンツをホスティングしていた。このソースは、https://github.com/nmemoto/nmemoto.dev/tree/main/home で管理している。
https://nmemoto.dev のコンテンツは [Git連携](https://developers.cloudflare.com/pages/get-started/guide/#connect-your-git-provider-to-pages)を使ってビルドデプロイするように構成していた。

## 調査・検討・対応

### Cloudflare Pagesのデプロイ方法

https://developers.cloudflare.com/pages/get-started/guide によると、Cloudflare Pagesのデプロイ方法は以下の3つあると記載されている。

- Git連携(Connecting your Git provider to Pages.)
- Direct Upload(Deploying your prebuilt assets right to Pages with Direct Upload.)
- Wrangler(Using Wrangler from the command line.)

Git連携はこの設定を行う際にビルド設定も行うことができるが、その他の2つはビルドされたアセットがある前提での実行を前提としている。

また、https://developers.cloudflare.com/pages/get-started/direct-upload/ によると、Direct Uploadの方法として[Wrangler](https://developers.cloudflare.com/pages/get-started/direct-upload/#wrangler-cli)と[Drag and Drop](https://developers.cloudflare.com/pages/get-started/direct-upload/#drag-and-drop)の2つの方法がある。実質的には以下の3つと私は考えている。

- Git連携(ビルド設定含む)
- Wrangler(ビルドは別途要検討)
- Drag and Drop(ビルドは別途要検討)

個人的な意見ではあるが、ビルド・デプロイに関する要件がなければ、ビルド処理をCloudflare側で行ってくれるGit連携の設定の手間が少ないGit連携を選定する。次点で、ビルドされたアセットさえあればCLIでデプロイを行えるWranglerでのデプロイ方法を検討する。Drag and Dropは手動操作なので継続運用していく際の通常運用手順としては選定しない。

### 1リポジトリで複数の Cloudflare Pages のアプリケーションをデプロイする場合

Cloudflare Pagesでプロジェクトを作成し、そのプロジェクトにGit連携する場合、すでにGit連携されているリポジトリは選択できないようになっていた。
つまり、すでにGit連携されているリポジトリ内で新しくCloudflare Pagesアプリケーションをデプロイしたい場合にそのアプリケーションのためのGit連携はできない。
それ故、この場合のWranglerを用いたデプロイの検討が必要となる。

### Wranglerでのデプロイコマンド

Cloudflare PagesアプリケーションのWranglerでのデプロイは、[wrangler pages deployコマンド](https://developers.cloudflare.com/workers/wrangler/commands/#deploy-1)の実行で行う。
デプロイの事前にプロジェクトを作成しておく必要があり、[wrangler pages project createコマンド](https://developers.cloudflare.com/workers/wrangler/commands/#project-create)の実行か、手動でのプロジェクト作成をする必要がある。

なお、[C3(create-cloudflare-cli)](https://developers.cloudflare.com/pages/get-started/c3/)を使うと、WebフレームワークをCloudflare上で動作させる前提の設定や構成を含んだプロジェクトを作成することができる。
(といっても、Cloudflare特有の設定は、Wranglerを用いた開発環境の起動やデプロイのためのコマンド・設定がpackage.jsonに最初から含まれている程度ではある。)
筆者は、以下のコマンドで Astro のプロジェクトを作成していた。

```shell
npm create cloudflare@latest -- playground --type webFramework --framework astro --deploy
```

通常のプロジェクトに加えられた処理は以下である。`npx astro add cloudflare -y` でライブラリ追加しているのと、package.jsonにpages関連のコマンドが追加されているのがわかる。

https://github.com/cloudflare/workers-sdk/blob/main/packages/create-cloudflare/src/frameworks/astro/index.ts

Wranglerのデプロイ実行コマンドがわからなければ、とりあえずC3でプロジェクトを始めるのが良いと考える。

### Wrangler GitHub Actionの利用

GitHubでリポジトリ管理しており、CI/CDでGitHub Actionsを使う場合、Cloudflareが提供している[Wrangler GitHub Action](https://github.com/cloudflare/wrangler-action)を使うと、Wranglerのセットアップを省力化できる。

Wrangler GitHub Actionを使った、AstroのプロジェクトをCloudflare Pagesにビルドデプロイするために構成した設定は以下となっている。
https://github.com/nmemoto/nmemoto.dev/blob/main/.github/workflows/push.yml

詳しい使い方は https://github.com/cloudflare/wrangler-action を確認して頂きたい。

また、この処理を行うには別途Cloudflareから、[API Token](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)と[Account ID](https://developers.cloudflare.com/fundamentals/setup/find-account-and-zone-ids/)の入手、それらを[GitHub Actionsのシークレットへ設定](https://docs.github.com/ja/actions/security-guides/using-secrets-in-github-actions)する必要があるため、注意されたし。

おそらく、wrangler.tomlが設定されていれば不要かもしれないが、Deploy実行時のコマンドに`--project-name`オプションを指定し、プロジェクト名の設定を今回はしている。

## まとめ

Cloudflare Pagesアプリケーションのデプロイについて調査した。1リポジトリで複数のCoudflare Pagesアプリケーションのデプロイする場合、Git連携設定時にすでにGit連携されているリポジトリを選択できない仕様からWrangler(または、Drag and Drop)でのデプロイを最低1つのアプリケーションで行う必要がある。GitHub ActionsでWranglerコマンドを実行する場合、[Wrangler GitHub Action](https://github.com/cloudflare/wrangler-action)を使うとWrangler自体のセットアップを行う必要がないため便利。