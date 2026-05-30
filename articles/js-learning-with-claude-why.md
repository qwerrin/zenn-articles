---
title: "【AI学習法】JavaScriptをClaudeに聞きながら学んだら、「なぜ動かないか」がわかるようになった話【JS編】"
emoji: "⚡"
type: "tech"
topics: ["JavaScript", "Claude", "初心者", "ダークモード", "フォームバリデーション"]
published: true
---

## TL;DR

- `addEventListener` の第2引数に `()` をつけると、関数を「呼び出した結果」を渡してしまう
- `event.preventDefault()` を入れるとネイティブバリデーションも止まる——JSでぜんぶ自前にする必要がある
- ダークモードのボタン切り替えは `classList.toggle` + `localStorage` の組み合わせ
- 上の3つ、全部「なぜ動かないの？」から入ったから今も覚えてる

## はじめに

前回のCSS編では、CSS変数・`clamp()`・ダークモードの仕組みを実装した。

今回はその続き。`@media (prefers-color-scheme: dark)` でOS連動のダークモードは動いていたが、「ボタンで手動切り替えしたい」という欲が出てきてJavaScriptに手を出した。それと、フォームのバリデーションをJSで制御しようとしたら思ってたより罠があった話も書く。

「なぜこのコードで動かないんだ」をClaudeに聞き続けた記録です。

## `addEventListener` の `()` 問題：関数を渡すのか、呼び出すのか

最初にやらかしたのはここだった。

ダークモードを切り替えるボタンを作ろうとして、こう書いた。

```javascript
const button = document.querySelector('#theme-toggle');
button.addEventListener('click', toggleDarkMode());  // ← NG
```

「動かない」じゃなくて「ページを開いた瞬間に1回だけ実行される」という挙動になった。クリックしても何も起きない。

Claudeに「なんで？」と聞いたら、

> `toggleDarkMode()` のように `()` をつけると、「この関数を呼び出した戻り値」を渡していることになります。関数を渡したいときは `()` をつけない——`toggleDarkMode` と書きます。

言われてみると当たり前なのに、一人では気づかなかった。「呼び出すか」「渡すか」という区別が、このとき初めて実感を持って理解できた。

```javascript
button.addEventListener('click', toggleDarkMode);   // ← OK：関数そのものを渡す
button.addEventListener('click', toggleDarkMode()); // ← NG：呼び出した戻り値を渡す
```

さらに「コールバック関数って何？」と聞いてみた。

> `addEventListener` のような関数は「高階関数」と呼ばれ、関数を引数に受け取ります。そこに渡される `toggleDarkMode` が「コールバック関数」です。受け取る側が高階関数、渡される側がコールバック——向きが逆になりがちです。

「受け取る側」「渡される側」という整理で、ようやく頭の中が整った。

## ダークモード切り替えボタン：`classList.toggle` + `localStorage`

`()` 問題を乗り越えて、実装した切り替えロジックはこれ。

```javascript
const button = document.querySelector('#theme-toggle');

button.addEventListener('click', toggleDarkMode);

function toggleDarkMode() {
  document.documentElement.classList.toggle('dark');

  const isDark = document.documentElement.classList.contains('dark');
  localStorage.setItem('theme', isDark ? 'dark' : 'light');
}

// ページ読み込み時にユーザー設定を反映
const savedTheme = localStorage.getItem('theme');
if (savedTheme === 'dark') {
  document.documentElement.classList.add('dark');
}
```

`classList.toggle('dark')` は、`dark` クラスがあれば外し、なければ付ける。CSS側では `.dark` クラスがついているときに変数を上書きする。

```css
.dark {
  --color-bg: #111827;
  --color-text: #f9fafb;
  --color-primary: #60a5fa;
}
```

「なぜ `localStorage` が必要なの？」と聞いたら、

> `localStorage` を使わないと、ページをリロードするたびにダークモードがリセットされます。ユーザーが設定したことを次回も覚えておくために保存します。

なるほど、状態の「永続化」という概念がここで登場した。JSを触り始めると「値を保存する場所」の話が繰り返し出てくる、という話もあわせて教えてもらった。

## フォームバリデーション：`event.preventDefault()` の罠

HTMLのフォームには `required` や `minlength` でネイティブバリデーションが使える。それで十分だと思っていた。

が、「エラーメッセージを日本語にしたい」「バリデーション失敗時にスタイルを変えたい」という欲が出てきてJSを使い始めたら、詰まった。

```javascript
form.addEventListener('submit', (event) => {
  event.preventDefault(); // ← これを入れた瞬間、ネイティブバリデーションが止まる
  // JS側でバリデーション処理を書く...
});
```

`event.preventDefault()` は「ブラウザのデフォルト動作（ページリロード）を止める」ために入れた。でも副作用として、ネイティブバリデーションも止まってしまう。

「じゃあJS側でゼロから書くしかないの？」と聞いたら、

> `event.preventDefault()` を使う場合は、ネイティブバリデーションに頼らずJS側でぜんぶ制御する設計にします。HTMLの `required` などは見た目の手がかりとしては残せますが、実際のチェックはJSで書き直すことになります。

ここで「どちらか一方に統一する」という設計の判断が必要だとわかった。混在させると挙動が予測しにくくなる。

自前バリデーションの例はこんな感じ。

```javascript
form.addEventListener('submit', (event) => {
  event.preventDefault();

  const name = document.querySelector('#name').value;
  const errorEl = document.querySelector('#name-error');

  if (name.trim() === '') {
    errorEl.textContent = '名前を入力してください';
    errorEl.hidden = false;
    return; // 早期リターンで後続処理を止める
  }

  errorEl.hidden = true;
  // バリデーション通過後の処理...
});
```

`return` で早期リターンするパターンも、「なぜここで `return` するの？」と聞いたら「エラーがあった時点で後続の処理を走らせないため」と即答してもらえた。説明されるまで「なんか動いてるからOK」で済ませていたやつだ。

## ハマったこと

### `min` と `minlength` は別物

```html
<!-- NG：テキスト入力に min を使っても効かない -->
<input type="text" min="2">

<!-- OK：テキストの最小文字数は minlength -->
<input type="text" minlength="2">

<!-- min は数値入力に使う -->
<input type="number" min="1" max="100">
```

「なんかバリデーションが効いてないな」と30分ほど悩んだ。`min` は数値の最小値、`minlength` は文字列の最小文字数——別の属性だった。

### `querySelector` で取れない要素をいじろうとするとエラー

```javascript
const button = document.querySelector('#not-exist');
button.addEventListener('click', doSomething); // ← TypeError: Cannot read properties of null
```

`querySelector` は要素が見つからないと `null` を返す。`null` に対して `.addEventListener` を呼ぶとエラー。「まずその要素が本当にあるか確認する」が基本だと教えてもらった。

```javascript
const button = document.querySelector('#theme-toggle');
if (button) {
  button.addEventListener('click', toggleDarkMode);
}
```

## まとめ

HTMLとCSSで「骨格」と「見た目」を作り、JavaScriptで「動き」を足す——この3層の分担が、実装してみて初めて実感できた。

今回のポイントをまとめると：

- `addEventListener` には関数を「渡す」（`()` をつけない）
- `event.preventDefault()` を使うなら、バリデーションはJS側に統一する
- ダークモードの永続化は `localStorage`
- `querySelector` の戻り値は `null` かもしれない

HTMLとCSSのときと同じで、「なぜ動かないの？」をそのまま聞けるのが一番よかった。エラーメッセージだけ見ても「何が起きているか」はわかっても「なぜそうなるか」がわからないことが多い。Claudeに聞くと、エラーの背景にある仕組みまで一緒に教えてもらえた。

次回は TypeScript 編として、型を使って「バグを実行前に防ぐ」体験について書く予定。

「AIをどう使えば学習になるんだろう？」と感じている方の参考になれば嬉しいです。Zennフォローいただけると、次回更新時にお知らせが届きます！

---

_2026年5月 学習開始2週目 / HTML/CSS/JS Day 8 完了_
