# Next.js 
- 参考: https://qiita.com/__knm__/items/32c632bbb93a107aca9e

## Next.js の主な機能
- プリレンダリング
  - SSR: Static Site Generation: 静的生成、ビルド時にレンダリングする
  - SG: Serverside Rendering: サーバーがわでレンダリングする
  - ISG: 
- ファイルベースルーティング（ダイナミックルート）
- API 作成（API Route）
- 開発者にやさしい開発環境（ゼロコンフィグ）

## ディレクトリ構成
- /pages
  - 配下のコンポーネントファイルが一つのページに対応する
  - pages/index.js = /
  - pages/about/index.js = /about
- /styles
  - CSS ファイルが格納される
- /public
  - 画像やフォントやアイコンなどの静的ファイル

## ルーティング手法
- 基本
  - ファイルまでのぱすがそのまま URL になる
  - ルーティング = URL で送信されたリクエストに対して、どのプログラムを実行するか紐づけること
- 動的ルーティング
  - /pages/blog/[number].js = /pages/blog/1 で
  - 同じ改装に固定の名称があれば、優先される。
  - useRouter フックで取得可能
```JavaScript
import { useRouter } from "next/router";

export default function Setting() {
  const router = useRouter(); // `query.name`などのプロパティを持つオブジェクトの格納
  return (
    <>
      <h1>{router.query.name}</h1>
    </>
  );
}
```
- useRouter 
  - push(): 画面遷移
```JavaScript
import { useRouter } from "next/router";
export default function Setting() {
  const router = useRouter();
  const clickHandler = () => {
    router.push("/"); // 引数に渡したurlに遷移する
  };
  return (
    <>
      <h1>{router.query.name}</h1>
      <button onClick={clickHandler}>トップへ</button>
    </>
  );
}
```
  - ドキュメント: https://nextjs-ja-translation-docs.vercel.app/docs/api-reference/next/router
- 一つのコンポーネントで複数画面を作成する方法
```JavaScript
import { useRouter } from "next/router";

const MultiPage = () => {
  const router = useRouter();
  const step = router.query.step ?? "0"; // null合体演算子
  const goToStep = (_step, asPath) => {
    router.push(`/multipage?step=${_step}`, asPath);
  };
  return (
    <div>
      {step === "0" && (
        <>
          <h3>Step: {step}</h3>
          <button onClick={() => goToStep(1, "/personal")}>Next Step →</button>
        </>
      )}
      {step === "1" && (
        <>
          <h3>Step: {step}</h3>
          <button onClick={() => goToStep(2, "/confirm")}>Next Step →</button>
        </>
      )}
      {step === "2" && (
        <>
          <h3>Step: {step}</h3>
          <button onClick={() => goToStep(0, "/multipage")}>Done</button>
        </>
      )}
    </div>
  );
};

export default MultiPage;
```
  - `const step = router.query.step ?? "0";` でクエリパラメータを取得している。
    - ?? は null合体演算子。左辺が null の時に、右辺を返す。

