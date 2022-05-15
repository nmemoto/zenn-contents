---
title: "GitHub TrendingをTUIで確認して選んだリポジトリのページをブラウザで開く gh-trending をつくった"
emoji: "👻"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["github", "go"]
published: true
---

# 概要

[Github Trending](https://github.com/trending)のリポジトリのデータを取得できるCLIツールが見当たらなかったため、GitHub CLI Extensionとしてgh-trendingを作成した。

https://github.com/nmemoto/gh-trending

[Github Trending](https://github.com/trending) のRepositoriesで表示されているリポジトリに関する情報をTUIで表示し、選択したリポジトリのページをブラウザで表示できるようにした。
Github Trending(のRepositories)では、Spoken Language(おそらくDescriptioonやREADME.mdなどの説明文で使用されている言語)、Language(プログラミング言語)、Date Range(期間)の3つの条件でリポジトリを表示できるが、このコマンド同様の条件での検索をオプションで指定できるようにした。


# 使い方

## インストール

[ghコマンド](https://github.com/cli/cli)を導入して、`gh extention install nmemoto/gh-trending`を実行すると、`gh trending`コマンドが使えるようになる。

## オプション

ヘルプは以下のとおり

```bash
$ gh trending --help
Check the Github Trending(https://github.com/trending) in the TUI and navigate to the repository page.

Usage:
  gh-trending [flags]

Flags:
  -h, --help                     help for gh-trending
  -l, --language string          Programming Language: go, typescript, ruby, .... anything is ok!
  -m, --mode string              Startup mode: browser(Select a trend repository and open its Github page) or json (default "browser")
  -p, --period string            Date Range: today, weekly or monthly (default "today")
  -s, --spoken-language string   Spoken Language: en(English), zh(Chinese), ja(Japanese), and so on.
```

# 仕組み

https://github.com/trending のリポジトリのデータ部分をスクレイピングして表示しているだけ。選択したリポジトリを[pkg/browser](https://github.com/pkg/browser)を使ってブラウザで開いている。

# まとめ

コマンドインターフェースでGitHub Trendingで表示されているリポジトリを確認し、選択したリポジトリのページをブラウザ表示するgh-trendingを作成した。

https://github.com/nmemoto/gh-trending

# 追記

作ったツールの問題に違いはないが、[manifoldco/promptui](https://github.com/manifoldco/promptui)のバグに起因するものや、表示もよくないところもあるので、これから見た目や使い勝手が大幅に変わってしまうかもしれない。
あと、テストも作らなくては。有識者にアドバイス頂けると幸いです。