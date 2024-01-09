---
title: "全ての要素や疑似要素に対して box-sizing: border-box; を設定することは大事"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["css", "boxsizing", "borderbox"]
published: true
---

## tl;dr

グローバル設定に以下を追加すべき。

```css
*, *::before, *::after {
  box-sizing: border-box;
}
```

これを設定しておくと、ページ全体で予測可能で管理しやすいボックスサイジングモデルとすることができる。

## box-sizingプロパティとは

box-sizingはCSSのプロパティで、要素の全体の計算方法を設定する。

https://www.w3.org/TR/css-sizing-3/#box-sizing や https://developer.mozilla.org/ja/docs/Web/CSS/box-sizing に記載されているので確認されたし。

設定値には、`content-box`と`border-box`の2つがある。
`content-box`はサイズ計算にボックスの内側を使う。つまり、サイズ計算にmargin/border/paddingを含まない。
`border-box`はサイズ計算にボックスの境界線内を使う。つまり、サイズ計算にborder/paddingを含む。（marginを含まない。）

## box-sizing: border-box; を使うべき

### box-sizing: content-box; の表示

https://developer.mozilla.org/ja/docs/Web/CSS/box-sizing#%E8%A9%A6%E3%81%97%E3%81%A6%E3%81%BF%E3%81%BE%E3%81%97%E3%82%87%E3%81%86 の2段目の設定値の表示は以下。Child ContainerがParent Containerからはみ出ている。

![](/images/box-sizing-border-box-is-important/box-sizing-content-box-demo.png =750x)

2段目の設定値には以下が設定されている。

```css
box-sizing: content-box;
width: 100%;
border: solid #5B6DCD 10px;
padding: 5px;
```

`width: 100%;`はサイズ計算にborder/paddingを含まずコンテンツそのものに設定しているため、paddingやborderを設定した分要素がはみ出している。

### box-sizing: border-box; の表示

https://developer.mozilla.org/ja/docs/Web/CSS/box-sizing#%E8%A9%A6%E3%81%97%E3%81%A6%E3%81%BF%E3%81%BE%E3%81%97%E3%82%87%E3%81%86 の3段目の設定値の表示は以下。Child ContainerがParent Containerに収まっている。

![](/images/box-sizing-border-box-is-important/box-sizing-border-box-demo.png =750x)

3段目の設定値には以下が設定されている。

```css
box-sizing: border-box;
width: 100%;
border: solid #5B6DCD 10px;
padding: 5px;
```

`width: 100%;`はサイズ計算にコンテンツそのものとborder/paddingを含んでいるため、Child Containerのborderの外側がParent Container内に収まっている。

これ故、`box-sizing: border-box;`としておくと、予測可能で管理しやすいボックスサイジングモデルとすることができる。

## box-sizingプロパティのデフォルト設定

にも関わらず、`box-sizing: content-box;`がデフォルト設定となっている。これはCSS2.1で規定されている振る舞いであるためとのこと。
https://www.w3.org/TR/css-sizing-3/#valdef-box-sizing-content-box に記載があった。

## 行うべき設定

上記より、以下のように全ての要素や疑似要素に対して`box-sizing: border-box;`を設定すべきである。

```css
*, *::before, *::after {
  box-sizing: border-box;
}
```

## 蛇足

記事を書き始めた後に、ChatGPTに聞いたら淀みなく答えてくれてしまった。
https://chat.openai.com/share/79ebae2f-63a8-41e6-a964-4b1de81c1816