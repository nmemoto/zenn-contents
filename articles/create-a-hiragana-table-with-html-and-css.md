---
title: "ひらがな表をHTMLとCSSで作成して、押したらWeb Speech APIでその1音を鳴らす"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["HTML", "CSS"]
published: true
---

ひらがな表(カタカナ表)のかなを押すと、Web Speech APIでその1音を鳴らすページを作成した。Google ChromeとFirefoxで動く。
https://0jscfu.csb.app/

ソースは以下で確認できる。
https://codesandbox.io/s/serene-resonance-0jscfu?file=/index.html

このページの作り方のメモを残す。

## あいうえおを縦に表示する

あ〜お、か〜こ、・・・の行の単位でpタグで囲い、そのpタグの内容を上から下に表示するために、`writing-mode: vertical-rl;`を設定した。

```css
p {
  writing-mode: vertical-rl;
  margin: 0%;
}
```

`writing-mode`については以下を参照のこと。
https://developer.mozilla.org/ja/docs/Web/CSS/writing-mode


## 行を右から左に表示する

すべての行(あ行からわ行まで)を右から左に表示するために、それらの行(pタグ)をdivタグで囲い、`display: flex; flex-direction: row-reverse;`を設定した。
divタグのクラス名を`kana`とした。

```css
.kana {
  display: flex;
  flex-direction: row-reverse;
}
```

## 真ん中に寄せる

`flex-direction: row-reverse;`を設定すると右に寄ってしまう。真ん中寄せにしたいので、`kana`に`justify-content: center;`を設定した。

```css
.kana {
  display: flex;
  flex-direction: row-reverse;
  justify-content: center;
}
```

## Web Speech APIで音声を出す

scriptタグに以下を設定した。かな一文字毎をaタグで囲い、aタグの要素毎にクリックして音を出すイベントリスナーを設定した。

```javascript
window.addEventListener("DOMContentLoaded", function () {
  const synth = window.speechSynthesis;
  let kanaList = document.getElementsByTagName("a");
  kanaList = Array.prototype.slice.call(kanaList);
  kanaList.forEach((kana, index) =>
    kana.addEventListener("click", function () {
      console.log(`${kana.textContent}`);
      const utterThis = new SpeechSynthesisUtterance(kana.textContent);
      utterThis.onend = function (event) {
        console.log("SpeechSynthesisUtterance.onend");
      };
      utterThis.onerror = function (event) {
        console.error("SpeechSynthesisUtterance.onerror");
      };
      utterThis.pitch = 1;
      utterThis.rate = 1;
      utterThis.lang = "ja-JP";
      synth.speak(utterThis);
    })
  );
});
```

Web Speech APIについては以下を参照のこと。
https://developer.mozilla.org/ja/docs/Web/API/Web_Speech_API
