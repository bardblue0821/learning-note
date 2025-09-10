# React 
- https://qiita.com/fujithuro/items/a17767e77866c2c73ed7
## DOM と 命令型プログラミング と 宣言型プログラミング
- DOM
  - HTML のオブジェクト表現
  - 木構造をしている
  - JavaScript から操作することができる
    - 特定の要素を増やす
    - レイアウトを変更する
```HTML
<script type="text/javascript">
  const app = document.getElementById('app');
  const header = document.createElement('h1');
  const text = 'Develop. Preview. Ship.';
  const headerContent = document.createTextNode(text);
  header.appendChild(headerContent);
  app.appendChild(header);
</script>
```
    - id=app という要素に h1 を追加する命令型プログラミングの例
      - <h1> は要素ノード、text はテキストノードと呼ばれる
    - 宣言型プログラミングは表示したいものだけを指示する
```JavaScript
import React from 'react';

function App() {
  return (
    <div id="app">
      <h1>Develop. Preview. Ship.</h1>
    </div>
  );
}

export default App;
```
## プロパティ
```JavaScript
function HomePage() {
  return (
    <div>
      <Header title="React" />
    </div>
  );
}
```
↓ title="React" が引き渡される
受け取り方①
```JavaScript
function Header({ title }) {
  console.log(title);
  return <h1>{title}</h1>;
}
```
受け取り方② ドット表記
```JavaScript
function Header(props) {
  return <h1>{props.title}</h1>;
}
```
受け取り方③ テンプレートリテラル（テンプレート文字列）
（Python でいう、文字列リテラル）
```JavaScript
function Header({ title }) {
  return <h1>{`Cool ${title}`}</h1>;
}
```
受け取り方③ 関数
```JavaScript
function createTitle(title) {
  if (title) {
    return title;
  } else {
    return 'Default title';
  }
}
 
function Header({ title }) {
  return <h1>{createTitle(title)}</h1>;
}
```
受け取り方④ 三項演算子
```JavaScript
function Header({ title }) {
  return <h1>{title ? title : 'Default Title'}</h1>;
}
```

リストで表示する場合
```JavaScript
function HomePage() {
  const names = ['Ada Lovelace', 'Grace Hopper', 'Margaret Hamilton'];
 
  return (
    <div>
      <Header title="Develop. Preview. Ship." />
      <ul>
        {names.map((name) => (
          <li key={name}>{name}</li>
        ))}
      </ul>
    </div>
  );
}
```


## イベントハンドリング
```JavaScript
function HomePage() {
  // 	...
  function handleClick() {
    console.log('increment like count');
  }
 
  return (
    <div>
      {/* ... */}
      <button onClick={handleClick}>Like</button>
    </div>
  );
}
```
Like ボタンが押されると onClick が発火し、function handleClcik() が呼び出される

## ステート と フック
```JavaScript
function HomePage() {
  // ...
  const [likes, setLikes] = React.useState(0);
 
  function handleClick() {
    setLikes(likes + 1);
  }
 
  return (
    <div>
      {/* ... */}
      <button onClick={handleClick}>Likes ({likes})</button>
    </div>
  );
}
```
- useState は React が定義する関数。
- 一つ目を状態変数、二つ目をそれを変化させる関数とする。

## ネットワーク境界
- Next.js では サーバーコンポーネント と クライアントコンポーネント をサポートする
- ネットワーク境界はそれを分離するための概念的な境界線
- クライアントコンポーネント
  - 従来の React 部分。UI など。
  - useState などはクライアント側。
  - 既定では React ファイルがサーバーコンポーネントとして扱われるため、ファイルの始めに'use slient' という文字列が必要。
- サーバーコンポーネント
  - Node.js のようなサーバーサイドで実行される。
  - データベースやファイルシステムにアクセス可能
  - 静的なコンテンツやサーバーサイドで処理を要する重い処理など
  - レンダリングされると、 React Server Component Payload (RSC ペイロード) と呼ばれる特殊なでーたがクライアントに送られる。
    - RSCペイロード
      - サーバーコンポーネントのレンダリング結果
      - クライアントコンポーネントがレンダリングされる箇所のプレースホルダ
  - 既定ではサーバーコンポーネントが使用される



