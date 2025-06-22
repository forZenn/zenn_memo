---
title: "TypeScriptの強力な機能たち：なぜそれらを使うのか？"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["contest2025ts", "ts", "typescript"]
published: true
---

「なぜこのtypescriptの機能を使うのですか？」と質問された際に、
自信をもって明確に説明できるようになるための記事です。TypeScriptを学習する上で
多くの人が疑問に感じる点について、著者の経験も踏まえながら解説していきます。

TypeScriptの本質は、堅牢なドキュメンテーションと強力なリンター機能にあると著者は考えています。
JavaScriptとは異なる上、独特な概念も多いので、他の言語に慣れ親しんだ人にとっては戸惑うこともあるでしょう。

入門書を読んでも「これ、一体何に使うの？！」と疑問符が浮かぶような機能も存在します。
本記事では、そうした機能の具体的な利用シーンやメリットについて深掘りします。

特に断りが無い限り一緒に書いているjsdocとコード上、linter上全く同じように動作します。

## バージョン情報

2025-06-18日時点での情報に基づいています。

TypeScript 5.8.3

## 環境構築

今回のTypeScriptの機能を確認するにあって、
nodejs,deno、ts-nodeなどの言語、ツールがインストールされていない場合は
あらかじめインストールしておいてください。

### windows

```powershell
# nodejs のインストール
winget install OpenJS.NodeJS.LTS

# グローバルにts-nodeをインストールしておく。
npm install -g ts-node

# denoのインストール
winget install DenoLand.Deno
```

### macos

```bash
# nodejsのインストール
brew install nodejs

# グローバルにts-nodeをインストールしておく。
npm install -g ts-node

# denoの場合
brew install deno
```

## REPL環境での制約について

REPL環境では多くの場合、TypeScriptトランスパイラの厳密な型チェックが行われません。

例えば、下記のようにtypescriptとしてエラーを出すべきコードはREPL環境ではエラーにならないのです。

```ts
// 文字列に数字を入れてもエラーにならない。
const permutation: string = 1234;

// readonlyな配列の要素に再代入してもエラーにならない。
const countDown: readonly number[] = [1, 2, 3]
aaa[0] = 5;

// インデックスシグネチャを使っているわけでもないのに、
// 動的にキーを追加してもエラーにならない。
interface Point {
    x: number,
    y: number
}

const point: Point = {x: 30, y: 40};

point.z = 40;
```

REPL環境では、TypeScriptの型アノテーションや構文が、実質的にJavaScriptの単なるコメントとして扱われるか、非常に限定的な構文チェックのみが行われます。これにより、TypeScriptが提供する性的型安全性が損なわれ、開発者が意図しないバグを見逃す可能性が高くなります。

そのため、今回紹介する文法や機能についてチェックする場合は、
下記のようにts-nodeやdenoコマンドを使って、スクリプトファイルとして動作確認することをお勧めします。

```bash
# ts-nodeはtsconfig.jsonがあるプロジェクト直下ではなくて、全くないところでやること。
ts-node 実行したいファイル.ts
deno --check 実行したいファイル.ts
```

## 基本

ここからはTypeScriptの基本的な機能について解説します。すでにご存知の方は
読み飛ばして頂いて構いません。

### 型定義

TypeScriptではtypeキーワードを使用して新しい型を定義できます。

```ts
type hello = string
```

JSDocでの表記は以下のようになります。

```ts
/** @typedef {string} hello */
```

実際の値から型を取得して型定義を作成したい場合は下のようにtypeofを使って、値から型を取得します。

```ts
const hello = "Hello World";

// Helloはstring型と推論される。
type Hello = typeof hello;
```

JSDocでの表記は以下のようになります。

```ts
const hello = "Hello World";

// Helloはstring型と推論される。
/** @typedef {typeof hello} Hello */
```

配列及びタプル、オブジェクトリテラルから型を取得する場合は下記のように
キーとして使われている型を指定します。

```ts
const countDown = [1, 2, 3];

// CountDown型はnumber型と推論される。　
type CountDown = typeof countDown[number];


const invoiceMessage = {
    success: "請求書を作成しました。",
    error: "請求書の作成に失敗しました。"
}

// キーは固定値なので、InvoiceStatusは"success" | "error"と文字列リテラルのUnion型と推論される。
type InvoiceStatus = keyof typeof invoiceMessage;

// InvoiceMessageの値は全てstring型なので、invoiceMessageはstring型と推論される。
type InvoiceMessage =  typeof invoiceMessage[keyof typeof invoiceMessage];
```

JSDocでの表記は以下のようになります。

```ts
const countDown = [1, 2, 3];

/**
 * CountDown型はnumber型と推論される。　
 * @typedef {typeof countDown[number]} CountDown
 */

const invoiceMessage = {
    success: "請求書を作成しました。",
    error: "請求書の作成に失敗しました。"
}

/**
 * キーは固定値なので、InvoiceStatusは"success" | "error"と文字列リテラルのUnion型と推論される。
 * @typedef {keyof typeof invoiceMessage} InvoiceStatus
 */

/**
 * InvoiceMessageの値は全てstring型なので、invoiceMessageはstring型と推論される。
 * @typedef {typeof invoiceMessage[keyof typeof invoiceMessage]} InvoiceMessage
 */
```

配列に複数の型が入っている場合は、後述するUnion型として推論されます。

```ts
const result = ["success", new Error("管理者に問い合わせてください")];

// Result型はstring | Error型のUnion型と推論される。
type Result = typeof result[number];
```

JSDocでの表記は以下のようになります。

```ts
const result = ["success", new Error("管理者に問い合わせてください")];

/**
 * // Result型はstring | Error型のUnion型と推論される。
 * @type {typeof result[number]} Result
 */
```

他の言語経験者からすると、これがドキュメンテーションなのかコードなのか判別しづらく、
違和感を覚えるかもしれません。しかし、TypeScriptではこのような記述が頻出するので、
徐々に慣れていきましょう。

### interface

interfaceは、TypeScriptでオブジェクトの構造を定義する際によく利用されます。typeとの
使い分けで迷うこともあると思いますが、基本的な考え方としては、「オブジェクトの具体的な構造を
定義する際はinterface、それ以外の型\(プリミティブ型、Union型、Tuple型\)などを定義するときは
typeと覚えておくと良いでしょう。

技術的な仕様の違いとしては、interfaceはextendsによる拡張が可能である点が挙げられます。

interfaceは以下の2つの主要な目的で使われます。

1. クラスのインスタンスの型定義：　クラスがどのようなプロパティを持つかを定義します。
2. オブジェクトリテラルの型定義；　特定の構造を持つオブジェクトがどのようなプロパティを持つかを定義します。

他の言語経験者からすると、TypeScriptのinterfaceの概念が異質に感じられるかもしれませんが、
慣れることが重要です。


```ts
interface Point {
    x: number;
    y: number;
}
```

JSDocでの表きは以下のようになります。

```ts
/**
 * @interface {Object} Point
 * @propterty {number} x
 * @propterty {number} x
 * /
```

### TypeScriptの「バグ」とは何か？

「TypeScriptは型があるから、曖昧さがなく安全!」という言葉をよく耳にしますが、この
「安全」とは具体的に何を指すのでしょうか？時には「意味が無い」と感じるような、
ごく個人的なか着心地の問題として語られることもあります。

筆者の考えるJavaScriptにおける最大のバグの温床は、
**型が無いことではなく、オブジェクトにキーを動的に追加できること**にありました。
これに加えて、判別可能なUnion型\(後述\)を用いる習慣がなかったため、
未熟なプログラマーはいくらでも壊れやすいコードを書いてしまう状況にありました。

TypeScriptは、このような動的なキー追加による予期せぬ挙動をコンパイル時に
検知し、未然に防ぐことでコードの安全性を高めます。

### リテラル型

リテラル型はTypesScriptにおける必須知識の一つです。

TypeScriptでは、1.12や"Hello world"、trueといった特定の具体的な値そのものを型として
扱うことができます。これはnumber、string、booleanといったプリミティブ型のうち、
指定した値しか受け取れない型を意味します。

JavaScriptの「オブジェクトリテラル」と名前が似ているため、紛らわしいかもしれませんが、
「リテラル」とはプログラミング言語で「具体的な定数」を指します。CSharp, Java,Pythonなど
多くの言語で使われる一般的な用語です\(ただし、「リテラル型」という概念はTypeScript独自です。\)。

とりあえず、「リテラル型という特殊な型が存在するんだな」という理解で大丈夫です。

```ts
type Hello = "HelloWorld"

// Hello型は"HelloWorld"という文字列リテラルしか受け付けない。
const aaa: Hello = "HelloWorld";
```

JSDocでの表記は以下のようになります。

```js
/**
 * @typedef {"HelloWorld"} Hello
 */

/** @type {Hello} こんな感じでかける。 */
const aaa = "HelloWorld";
```

リテラル型だけだと一種類の値しか扱えないので、何が嬉しいのか全く分からないかもしれません。
しかし、これは後述のUnion型やas constと組み合わせることで、その真価を発揮します。

### Union型

Union型は、TypeScriptで非常によく使われる機能です。

「型のOR\(||\)と覚える」と良いでしょう。

TypeScriptは**単なる値であるリテラルも型として扱える**ため、
Union型は下のように列挙型\(enum\)の代わりとして使われることが多いです。

```ts

type OS = "Windows" | "MacOS" | "Linux";

// エラーが出る。 Type '"BSD"' is not assignable to type 'OS'
const yourOS: OS = "BSD";

```

nullableな型\(Null許容型\)は下のように表現します。

```ts
type NullableString = string | null;
```

また、undefinedを許容する場合は次のようになります。

```ts
type OptionalNumber = number | undefined;
```

JavaScriptではnullとundefinedの使い分けが曖昧な側面がありましたが、TypeScriptでは型安全性と意図の明確化のために、両者の意味合いを区別して使用することが推奨されます。
具体的には、変数が意図的に「値がない」状態であることを示す場合はnullを、変数がまだ値が割り当てられていない、または明示的に「未定義」であることを示す場合はundefinedを用いるのが一般的です。

### as \(castまたはType Assertion \)

TypeScriptでは、他の言語と同様に**Type Assertion \(as\)**を使って、
変数に強制的に型を割り当てることができます。これは、開発者がTypeScriptトランスパイラよりも
型について詳しい場合に、その知識をトランスパイラに伝えるために使用します。


```ts
// 状況的にこの型以外ありえないため、型アサーションを使用します。
let value: any = "This is a string";
let length: number = (value as string).length;
```

### 固定長の配列\(Tuple型\)

TypeScriptのTuple\(タプル\)型は、固定の長さと、各インデックスに異なる型を持つ配列を
定義するために使われます。これは特に、複数の異なる型の値を順序立てて扱う場合に非常に
便利です。

```ts
// 各々のインデックスに特定の型を持つ固定長の配列を定義。
type Point3D = [number, number, number];

const p: Point3D = [1,2,3]; // OK

// const q: Point3D = [1, 2]; // エラー:　長さが異なる。
// const r: Point3D = [1, 2, "3"] // エラー：　型が異なります。
```

皆さんお馴染みのReactのuseStateフックにも、このTuple型が使用されています。
useState()は、getter\(現在のstateの値\)とsetter\(stateを更新する関数\)の
2つの要素を持つTupleを返します。

```ts
// countはnumber型、setCountはReact.Dispatch<React.SetStateAction<number>>型と推論されます。
const [count, setCount] = useState<number>(0);
```

JSDocでの表記は以下のようになります。

```js

/** @type {[number, React.Dispatch<React.SetStateAction<number>>]} */
const [count, setCount] = useState(0);


// 初期値がない場合、booleanとundefined両方がありうるため、下のようになる。
/** @type {[boolean | undefined, React.Dispatch<React.SetStateAction<boolean | undefined>>]} */
const [flag, SetFlag] = useState();
```

### 制御文

TypeScriptコンパイラは、ifやelse, switchによる制御文により、
その時点での変数の型を自動的に絞り込みます。
これを**型ガード**と言います。

```ts
function processValue(value: string | number) {
    if (typeof value === 'string') {
        // このブロック内では、valueはstring型として扱われます
        console.log(value.toUpperCase());
    } else {
        // このブロック内では、valueはnumber型として扱われます
        console.log(value.toFixed(2));
    }
}
```

型ガードに関しては、詳細は型ガード関数と合わせて後述します。

### 型とinterfaceのimport export

#### type

typeキーワードで定義された型は、export typeおよびimport typeを使って、
export, importできます。 これにより、実行時のコードに影響を与えることなく、型情報のみをやり取りできます。

また、typeはコンパイル後にはJavaScriptとして出力されずに削除される。

exportの場合は下記のようになります。

```ts
type MyType = string | number;

export {
    type MyType
}
```

importの場合は下記のようになります。

```ts
// MyType.tsから型をimport
import type { MyType } from './MyType';

const value: MyType = "hello";
```

jsdocでimportを書く場合は以下のようになります。

```ts
// MyType.tsから型をimport
// jsdocでは@importはtype-chekingのみ提供する。[jsdoc import](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#import)
/** @import {MyType } from './index' */

/** @type {MyType} */
const value = "hello";
```

#### interface

interfaceは、typeと同様にTypeScriptのコンパイル後にはJavaScriptのオブジェクトに変換されす、削除されるが、
通常のJavaScriptのimport/exportと全く同じ表記でimport/exportできる。

```ts
interface MyInterface {
    name: string;
    age: number;
}

export {
    MyInterface
}
```

```ts
// MyInterface.tsからimport
import { MyInterface } from './MyInterface';

const user: MyInterface = { name: "Alice", age: 30 };
```

### class

classはTypeScript固有の機能ではなく、JavaScriptの機能ですが、
その使い所について誤解されがちです。
ここでは、classが有用となる場面いついて解説します。

classは、以下の状況で利用を検討すると良いでしょう。

1. オブジェクトのライフサイクル管理：　オブジェクトの生成から破棄までの過程で初期化、リソース解放などを行いたい時
2. リソースの解放：　ファイルディスクリプタ、データベースコネクション、ネットワークソケットなど、明示的な開放が必要なリソースを管理する場合。CSharpだとusingで開放するべきものたち。
3. オブジェクトの状態管理：　複数の関連するデータと、そのデータを操作するメソッドを一つの単位としてまとめ、複雑な状態を持つオブジェクトを管理する場合。

フロントエンド開発では、これらの状況が発生することは稀です。
状態管理に関しても、多くの場合は、クロージャで事足りてしまうため、classのような複雑な構造を使う必要があるときは
ほとんどありません。フロントエンドは非同期通信が基本であり、Promiseのcatchやfinallyで通信の成否に
応じたエラー表示を行うだけで済むことが多いでしょう。

「クロージャなんてあまり使わない!」と思うかもしれませんが、実際には多くのライブラリが
クロージャを意識せずに使えるように配慮されています。例えば、ReactのuseStateフックのset関数は
クロージャですが、これにより十分な状態管理が実現できています。

もし、あなたがここでclassを使おうと思ったのなら、その理由を自問自答してみてください。
おそらく、「データとそのデータを操作する関数を一つのまとまりとして扱いたい」と考えたからではないでしょうか？
あなたがJava系の言語出身者であれば、classやstructでデータを表現し、その値を関数で加工し、他の関数に参照の値を渡し、保存するという習慣がついているためclassを選択しようとしたのかもしれません。

しかし、データを加工したり、他の関数に渡したり、保存したりする処理に、ライフサイクル管理やリソース開放が必要でしょうか？おそらく必要ないでしょう。Javaは全てがclassである(一部のプリミティブなものを除く)ためclassを使う必要がありますが、そうでないならば、**オブジェクトリテラルと関数\(構造体+関数\)で実装する方が、シンプルかつ変更に強いコードになる**ことが多いです。

JavaScriptにおけるオブジェクトリテラルは、まさに「構造体」としての役割を果たします。

これまでの内容をバックエンド開発に当てはめて考えてみると、リソースの開放が頻繁に登場するため、
バックエンドではclassが必要となるコードが多くなることが理解できるでしょう。

リソースの解放とはdestructorに書いたり、
csharpでいうusingで自動的に解放されるような
処理である。

具体的に言えば、
ソケットやらdbやらのコネクションの接続やら、ファイルのクローズやら
メモリ解放などである。

お察しの通り、フロントエンド側でこれらを握ることは無く、
状態管理があってもクロージャで事足りるので、classなんて複雑なものを使う必要ない。
フロントエンドは非同期通信が基本でpromiseに対してcatch, finallyで
通信が上手くいったかどうかでエラーを表示するとかそれで済むはずだ。

クロージャなんてあんま使わない！と思うかもしれないが、ライブラリ側で
クロージャを意識しないで使えるように配慮されていて、
みんなが使っているreactのuseStateも値をsetする関数の方はクロージャだが、
それで十分足りていると思う。

ここでなぜあなたがclassを使おう!と思ったのか
自分で考えてみよう。

恐らく、データ+関数という形で値を処理したいからだと思う。
ここであなたががjava系の言語出身者なら
classかstructでデータを表現して、その値を関数で加工する、他に値を渡す、
保存するということをする習慣がついているから、
classを使おう!と思ったのである。

ところで、加工する、他に値を渡す、保存するという
処理にライフサイクルやリソースの解放が必要であろうか？
恐らくないだろう。
Javaは全てがclassなので、classを使う必要があるが、
そうでなければ、
構造体+関数で実装する方がシンプルかつ変更に強い。

jsで構造体があったけ？と
思うかもしれない。
それはオブジェクトリテラルがその役割を果たしてくれる。

上までの内容に対して、今度はバックエンドで考えてみると
わかるが、リソースの解放が出てくるので、
クラスが必要なコードが多く出てくる。

ここからは中級者むけの話題なので興味ない人は読み飛ばしてください。

ここでdenoの実装を見て見てみよう!
denoのext配下にはDenoでデフォルトでimportされている
ライブラリが書かれており、ここにjsから呼ぶ部分と
rustからの実装と両方が書かれている。
構成としては下のようになっている

libc->rustでjsから呼ぶ処理を記述(ファイルの場合はファイル処理など具体的な処理)-> jsに紐づけられている
Deno.FsFileで呼び出せるという仕組み。
ビルドしたらrustからffiとしてjsから呼び出せるということだ。

js側ではDisposeでrust側のリソースを解放すれば、良い。
jsは

ファイルの実装

https://github.com/denoland/deno/blob/main/ext/fs/30_fs.js


## 応用

ちゃんとコード書くなら知っておくべきこと。

### discriminated union\(判別可能なUnion\)

筆者は「判別可能なUnion型」という訳はあまり良くないと思いますし、英語の命名自体もそのUnion型よりも
パターンに注目して「Discriminated Union Pattern」と命名すべきだったんじゃないかな？と思います。
しかし、この名前が定着しているため、この表記で以下の文章は続きます。

ここについては先にreadonlyやインデックスシグネチャについての説明を読んでから、読むことをお勧めします。

Union型に制御構造を持たせようとすると、自然とこの書き方にだどりつきます。

TypeScriptがない純粋なJavaScriptのみのプロジェクトでは、以下のようなコードが多く見られました。

```js

// 状態に合わせたスタイルの作成
function getButtonStyle(buttonState) {
  let style = {};

  if (buttonState.type === 'loading') {
    style.backgroundColor = 'lightgray';
    // 'spinnerColor' は loading 状態にしかないプロパティ
    style.color = buttonState.spinnerColor; // 間違って textColor などと書いてしまうと undefined
  } else if (buttonState.type === 'error') {
    style.backgroundColor = 'pink';
    // 'textColor' は error 状態にしかないプロパティ
    style.color = buttonState.textColor;
  } else {
    style.backgroundColor = 'lightblue';
    style.color = 'black';
  }
  return style;
}
```

このコードでは、パッとみただけではどのようなオブジェクトリテラルが返されるのかが分かりにくいでしょう。
これくらいの規模であれば問題ないかもしれませんが、ロジックが複雑化し、改修が繰り返されると、
ソースコードを追うのは不可能になります。

そのため、筆者も以下のようなコメントを追加することがよくありました。想定しているオブジェクトリテラルの
パターンをコメントで書かないと、何が返ってくるのか分からなかったからです。

```js
// ボタンの状態を表すオブジェクトとしては下の状態を想定している。
// ローディング状態: { type: 'loading', message: '読み込み中...', spinnerColor: 'blue' }
// エラー状態: { type: 'error', errorMessage: '操作に失敗しました', textColor: 'red' }
// 通常状態: { label: 'クリックしてください' }
```

この例ではtypeプロパティでボタンの状態を判別していますが、これでは制御文に従って動的に
プロパティを作成することになり、ソースコードを追うのが非常に困難で壊れやすくなります。

また、loadingが0、errorが1で、defaultは特に値を想定していない\(undefinedで良いと考えている\)といった酷いケースもあります。

さらに、呼び出し先の関数でプロパティを追加することも多々ありました。

```ts
// 例: 後から編集権限がないviewerの状態を追加する場合
// JavaScriptでは以下のように簡単にプロパティが追加できてしまう
const buttonState = { type: 'viewer' };
buttonState.canEdit = false; // TypeScriptではエラーになるが、JavaScriptではそのまま追加される
```

これらについては判別可能なユニオン型を用いることで、このような問題を解決することができます。

JavaScriptは参照の値渡しであり、CSharpのようにreadonly修飾子がないため、簡単に破壊的な
コードを書いてしまいがちです。

筆者も実際に、このようなコードがあちこちに散らばっているシステムの改修や機能追加を行ったことがありますが、非常に苦痛でした。
これらはJavaScriptの問題というより、プロジェクトにいるプログラマーのスキルレベルの問題\(コメントなどのドキュメンテーションでも回避可能\)が大きく関係しています。しっかりと設計すれば、判別可能なUnionという言葉やパターンを知らなくても、同じような解決策に辿り着くからです。

たまに、status、typeフラグを1, 2, 3のように数値で表現している人を見かけますが、これは非常に危険なのでやめるべきです。多くの場合、後から条件が増えていき、1, 2, 3がマジックナンバーと化します。

なぜこのような危険な書き方をしている人が多いのか不思議でしたが、『サバイバルTypeScript』に数値の例が書かれていました。

https://typescriptbook.jp/reference/values-types-variables/discriminated-union#ディスクリミネータに使える型

ディスクリミネータに使える数字として「1, 200」などというふうに説明されています。

数字で書く場合はHTTPのstatusコードのように、それ自体が実質的に判別可能なUnion型として機能しており、今後変更されないことが確定している、かつ多くの人が共通で使うもの以外は避けるべきでしょう。

例えば、次のような会員システムを考えてみてください。



後からその会員は２種類3種類にふえませんか？
無効な会員も退会した人間と休眠状態の会員と、
無料会員を1有料会員を2,無効な会員を3とした後に
Vip会員が増えませんか？それを4に割り当てたりしませんか？
ひどい場合は数字であることを利用して、
無効な会員を3以上として、

会員を判定するのに===3のように判定しており、
それに合わせて一部のスタイルを変更していたりしませんか？
無効な会委員が


### バリデーションライブラリ

TypeScriptの文法や機能ではありませんが、型に関する説明をする上で不可欠な要素なのでここで触れておきます。

ZodやJoiなどのバリデーションライブラリは、主にフォームの入力値やAPIからのレスポンスなど、外部から来る値のバリデーションに使用されます。

これらのライブラリがない場合、以下のような問題に直面します。

1. 頻繁にasや型ガードを書く必要が出る: 外部からの入力値の方が定まらないため、延々とasを使用したり、複雑な型ガードロジックを書いたりする必要が出てきます。
2. 複雑な型ガードの実装：　「有効なメールアドレスかどうか？」と行った複雑な条件を型ガードとして自分で実装するのは難しい。

バリデーションライブラリは、これらの手間を省き、型ガードやasを記述する量を減らすらためのライブラリと考えると良いでしょう。

### インデックスシグネチャ

インデックスシグネチャは、TypeScriptでオブジェクトリテラルに動的にキーを追加するための機能です。元々JavaScriptでは、オブジェクトリテラルに動的にキーやプロパティを追加できました。

```js
const point = {
    x: 30,
    y: 50,
}

// 元々無いキーを同的に追加できる。
point.z = 100;
```

しかし、TypeScriptではこのような動的なキー追加は通常禁止されており、明確にコンパイルエラーとなります。最近ではJavaScriptでも、エディタ上で黄色い下線で警告が表示されることがあります。


#### 動的なプロパティ追加を許可するには?

TypeScirptでもどうしても動的にキーを追加したい場合は、
インデックスシグネチャを使います。
これは、interfaceやtypeでオブジェクトの定義をお交際に、任意のプロパティの型を明示的に許可する方法です。

以下は例です。

```ts
// [慣習的にkeyをよく使う。: キーとして使える値(普通はstringを使う)]: 追加するキーの型
interface Point {
    x: number;
    y: number;
    [key: string]: number;
}

const point: Point = {
    x: 30,
    y: 40,
}

// 動的にプロパティをいくらでも追加できる。
point.z = 30;
point.w = 50;

//　ただしnumber型以外の値を代入するとエラーになる。
point.uu = "Unsigned Value"; // エラー：型 'string'を型'number'に割り当てることはできません。
```

**インデックスシグネチャはJSDocにこれに相当する表記はありません。**

「動的にキーを追加できる」というスタイルが当たり前になると、
TypeScriptの型安全性や静的解析の恩恵が失われてしまうという懸念もあります。

よって、アプリケーションコードでは、期待される値が決まっているため、通常は以下のように具体的なプロパティを定義します。

```ts
interface Point {
    x: number;
    y: number;
}

interface ThreeDObject extends Point {
    z: number;
    w: number;
}

const threeDObject: ThreeDObject = {
    x: 30,
    y: 40,
    z: 50,
    w: 30
}
```


### ジェネリクス

ジェネリクスとは、型を引数として取れる関数やオブジェクトのことです。
これにより再利用性の高いコードを作成できます。

ジェネリクスとしてどのプログラミング言語でも使われるのは
配列、集合、行列でしょう。

```ts
// 例: 配列の要素の型をジェネリクスで指定
function identity<T>(arg: T): T {
    return arg;
}

let output = identity<string>("myString"); // outputはstring型
let output2 = identity<number>(123); // output2はnumber型
```

JSDocでの表記は次のようになります。

```ts
/**
 * @template T
 * @param {T} arg 
 * @returns {T}
 */
function identity(arg) {
    return arg;
}

// それぞれ下記のように推論される。
let output = identity("myString"); // outputはstring型
let output2 = identity(123); // output2はnumber型
```

####　ジェネリクス型制約

しかし、アプリケーションコードの場合、ほとんどのケースでジェネリクスとに入る方は数通りに限定
されます。そのため、ジェネリクス型制約を使うことで、型をより厳密に縛ることが推奨されます。

型制約を使わずにジェネリクスを使うことはライブラリでは多々ありますが、アプリケーションコードでは
その頻度は少なくなります。

```ts
// ジェネリクス型制約は <T extends 型1 | 型2>という表記になる。
// Tはstringまたは配列のいずれかの型に限定されます
function getLength<T extends string | unknown[]>(item: T): number {
    return item.length;
}

getLength<string>("abc"); // 3
getLength<number[]>([1, 2, 3, 4]); // 4
```

JSDocでは下記のような書き方になります。

```ts
// ジェネリクス型制約は <T extends 型1 | 型2>という表記になる。
// Tはstringまたは配列のいずれかの型に限定されます
/**
 * @template {string | unknown[]} T stringまたはnumberの型を継承した型と縛ることができる。
 * @param {T} item 
 * @returns {number}
 */
function getLength(item) {
    return item.length;
}

getLength("abc"); // 3
getLength([1, 2, 3, 4]); // 4
```

他のプログラミング言語ではジェネリクスを使ったことはあっても、自分で実装した経験がない人も多いかもしれません。しかし、TypeScriptでは非常に頻繁に利用することになります。
具体例を挙げて理解を深めましょう。

UIコンポーネントでは、コンボボックスやMenuItemなど、リスト周りの処理でジェネリクスがよく使われます。これは、表示するデータの型が確定していない段階でコンポーネントを設計し、後から多様なニーズ（コンボボックスとして見せたい、MenuItemとして見せたいなど）に対応できるようにするためです。

ラジオボタン、セレクトボックス、チェックボックスなど、表示形式をギリギリまで決定しない必要がある場合にもジェネリクスは有効です。これにより、コンポーネントの差し替えが非常に楽になります。

MUIのRadio ButtonやCheckboxのドキュメントを見ると、ジェネリクスがどのように活用されているかを理解できるでしょう。

大抵の場合、継承で済むことが多いですが、より汎用性が求められる場合にジェネリクスを使います。主にライブラリでその真価を発揮するでしょう。

```ts
import Radio from '@mui/material/Radio';
import RadioGroup from '@mui/material/RadioGroup';
import FormControlLabel from '@mui/material/FormControlLabel';
import FormControl from '@mui/material/FormControl';
import FormLabel from '@mui/material/FormLabel';
```
https://mui.com/material-ui/react-radio-button/

https://mui.com/material-ui/react-checkbox/



### readonly

名称的にconstとreadonlyの違いについて疑問を持つ方もいるかもしれません。

constは「変数への再代入の禁止」を意味するのに対し、readonlyは「プロパティへの再代入の禁止」
を意味します。

readonlyは、classのプロパティやinterfaceのプロパティに適用できます。
interfaceに適用できるということは、オブジェクトリテラルの「プロパティへの再代入を防げる」ということです。

readonlyプロパティを初期化できるのは、以下のケースのみです。

1. クラス定義時の初期化
2. コンストラクタでの初期化
3. オブジェクトリテラルの初期化

```ts
interface Point {
    x: number;
    readonly y: number; // yは読み取り専用
}

const point: Point = {
    x: 30, 
    y: 40, // yはこの初期化時のみ値を変更できる。。
}

//　初期化以降に値を変えようとするとエラーになる。
point.y = 50; // エラー: 読み取り専用プロパティであるため、'y' に割り当てることはできません。
```

TypeScriptの興味深い機能として、配列、Set、Map、オブジェクトリテラル自体にも
readonlyを適用できる点です。この場合は、そのオブジェクトのプロパティの変更や追加が禁止されます。

```ts
// 配列の場合は、各インデックスに入っている値の変更ができなくなる。

const countDown: readonly number[] = [3, 2, 1, 0];
// 下記のように描いても同じ意味になる。
const countDown2: ReadonlyArray<number> = [3, 2, 1, 0];

// エラーになる。
// countDown[0] = 5; // エラー: 読み取り専用の型であるため、インデックスシグネチャに割り当てることはできません。

// pushなどの配列の要素を追加しようとする処理をするとエラーになる。
// エラーになる。
countDown.push(6) // エラー: 型 'readonly number[]' にはプロパティ 'push' がありません。


const abc: ReadonlySet<number> = new Set([1, 2, 3, 4]);

// addやdeleteをしようとするとエラーになる。
abc.add(5) // エラー: 型 'ReadonlySet<number>' にはプロパティ 'add' がありません。


// オブジェクトリテラルの場合は次のようになります。

interface User {
    name: string;
    password: string;
}

const aaa: Readonly<User> = {
    name: "パパス",
    password: "tonnura"
}
```

変数 : readonly 型 と表記を省略できるのは配列だけです。

特に、関数のオブジェクトの引数をreadonlyにできる点は覚えておきましょう。
たまに知らない人がいます。JavaScriptは参照の値渡しなので、関数の呼び出し先でプロパティなどの値を変更すると、
その変更が呼び出し元に反映されてしまいます。

以下に、readonlyを使わない場合の困った例を書きます。

```ts
interface Point {
    x: number;
    y: number;
}

function countDownFunc(count: number[]) {
    // 参照元のオブジェクトのプロパティが変更できてしまう。
    count[0] = 5;
}

function pointFunc(point: Point) {
    // 参照元のオブジェクトのプロパティが変更できてしまう。
    point.y = 34;
}

const countDown = [1,2,3];

countDownFunc(countDown); 

console.log(countDown); // [ 5, 2, 3 ]と表示。呼び出し先の関数でプロパティの値を変更できてしまう。　

const point: Point = {
    x: 1,
    y: 2
}

pointFunc(point);

console.log(point);  // { x: 1, y: 34 } 上と同様に変更できてしまう。
```

下記のように引数にreadonlyをつけると、
参照先の関数でプロパティの変更が禁止できるので
より安全に配列やオブジェクトリテラルを扱うことができます。

```ts
interface Point {
    x: number;
    y: number;
}

function countDownFunc(count: readonly number[]) {
    // 要素の変更がエラーになる。
    // count[0] = 5; // エラー: 読み取り専用であるため、変更できません。
}

function pointFunc(point: Readonly<Point>) {
    // プロパティを変更するのでエラーになる。
    // point.y = 34;
}
```

残念ながら**readonlyは値を変更してもJSDocではエラーになりません**ので、
TypeScirptではなく、JSDocを使う場合には注意が必要です。

```ts
/** @type {ReadonlyArray<number>} */
const abc = [1, 2, 3];

// エラーにならない。
abc[0] = 5
```

#### readonlyを基本的に必須にすべきか？

「exportしたり、公開するオブジェクトは基本的にreadonlyにすべきで、
そうでないコードはreadonlyを神経質に使う程でもない。
ソースコードの変更をおこうときに不便だし。」
って昔は思っていました。

しかし、JavaScriptは破壊的な処理を書きやすく、
簡単に壊すコードを書く人がそれなりにいることが分かり、
最近ではアプリケーションコードでも基本的にreadonlyを使うべきなのではないか？
と考えています。
時に関数の引数に関しては積極的にreadonlyを使うべきでしょう。

### as const

~~ややこしいから、なんか別の名前にして欲しかった。~~

なぜかインターネットで検索するとreadonlyとの比較がやたら多いですが、
as constの基本的な役割はコードから型を生成することであり、**readonlyとの比較は適切ではありません。**
用途が全く異なります\(昔はそういう使い方がメインだったとかあるかもしれませんが、筆者は昔の経緯は知りません。\)。

ライブラリを作ったり、型パズルを行う場合には必須の機能です。

例えば、下のような配列からOS型を作成したい場合を考えます。

```ts
const osNames = ["MacOS", "Windows", "Linux"];
```

as constを知らない場合あなたは恐らく、次のようにtypeofを使って型を
作成することを考えると思います。

```ts
const osNames = ["MacOS", "Windows", "Linux"];

type OS = typeof osNames[number];
```

しかし、最初のtypeofの説明でもしましたが、配列に対してtypeosfを行うとOS型はstring型であると推論されてしまいます。

```ts
// 上のように
type OS = string;
```

下記のようにtypeofから型を取得する変数を変更不可なreadonlyの配列として宣言しても結果は同じです。

```ts
// readonlyにより、各々のプロパティは変更不可になるが...
const osNames: readonly string[] = ["MacOS", "Windows", "Linux"];

// これでもやはりOS型はstring型と推論されてしまう。
type OS = typeof osNames[number];
```

これはreadonlyはあくまでプロパティの変更不可という制約を与えるだけで、
「値そのものを型とする」という推論の挙動とは直接関係がないためです。

どうすれば、配列の値から取得した型を"MacOS" | "Windows" | "Linux"のUnion型と推論させることができるでしょうか？

この問題を解決するために存在するのが、as constです。
このキーワードは**typescriptコンパイラに対してできるだけ具体的なリテラル型として推論**させるように指示します。
結果としてオブジェクトのプロパティに対して再起的にreadonly制約を課しますがそれは副次的なものです。

```ts
const osNames = ["MacOS", "Windows", "Linux"] as const;

// osNamesの各要素がリテラル型として推論され、それをUnion型として結合します。
type OS = typeof osNames[number];
```

上のコードは下記のようなUnion型の型宣言と同じ意味になります。

```ts
type OS = "MacOS" | "Windows" | "Linux";
```


オブジェクトリテラルも配列と同様に複数のリテラルを持つUnion型のとして取得するにはas constが必要になります。

```ts
const Invoice = {
    success: "請求書を作成しました。",
    error: "請求書の作成に失敗しました。"
} as const;

// "請求書を作成しました。" | "請求書の作成に失敗しました。"　というUnion型になる。
type InvoiceMessage = typeof Invoice[keyof typeof Invoice];
```

これらを聞いても全くありがたみが分からないかもしれませんが、
後述のenumの代わりによく使います。

下記がenumの代わりの例です。

```ts
const InvoiceEnum = {
    Success: "請求書を作成しました。",
    Error: "請求書の作成に失敗しました。"
} as const;

// "請求書を作成しました。" | "請求書の作成に失敗しました。"　というUnion型になる。
type Invoice = typeof InvoiceEnum[keyof typeof InvoiceEnum];

const invoice: Invoice = InvoiceEnum.Success;
```

#### 補足

as constをenumの代わりとして使う場合、[サバイバルTypeScript 列挙型の代替案2: オブジェクトリテラル](https://typescriptbook.jp/reference/values-types-variables/enum/enum-problems-and-alternatives-to-enums#%E5%88%97%E6%8C%99%E5%9E%8B%E3%81%AE%E4%BB%A3%E6%9B%BF%E6%A1%882-%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%83%AA%E3%83%86%E3%83%A9%E3%83%AB)では下記のように、Enumの方と型の方を両方、同じ名前で宣言しています。

```ts
const Position = {
  Top: 0,
  Right: 1,
  Bottom: 2,
  Left: 3,
} as const;
 
type Position = (typeof Position)[keyof typeof Position];
// 上は type Position = 0 | 1 | 2 | 3 と同じ意味になります
 
function toJapanese(position: Position) {
  switch (position) {
    case Position.Top:
      return "上";
    case Position.Right:
      return "右";
    case Position.Bottom:
      return "下";
    case Position.Left:
      return "左";
  }
}
```

これは一見名前が衝突してPositionのimport, export時にエラーが発生しそうに見えますが、
typescriptコンパイラは実行時に存在する値\(cost Positionオブジェクトの方\)とコンパイル後に消える型\(type Positionの方\)を異なる名前空間として扱うため、
名前の衝突が起きないのです。

実質的にenumとして使っているオブジェクトと型の名前を分けるかどうかは、
プロジェクトの方針に従いましょう。

今は大抵の場合は一致させているみたいです。

### Enum

TypeScriptにおけるEnumはたまに勘違いされるが、非推奨ではないです。
実際にtypescritコンパイラのソースコードにも使われています。

使い方的には他のプログラミング言語のenumを使ったことがある人ならすぐ使えるようになると思います。

```ts
enum ButtonColor {
    RED,
    GREEN,
    BLUE,

}; 

let color: ButtonColor = ButtonColor.RED;

console.log(color);
```

最近はtypescript界隈の特に理由がない場合はenumは使わない方が良いという主張が国内、国外問わず多いです。

一つは理由としてはEnumでやりたいことは下記のようにUnion型や上で説明したas constで表現できることです。

Union型を使って表現する場合

```ts
// シンプルにUnion型として使う。
type ButtonColor = "RED" | "GREEN" | "BLUE";
```

Union型だけだと、enumで表現したいキーとvalueが同じ値になってしまうので、
それを嫌うなら下のようにas constを使って実装します。

```ts
// as　constを使う場合。
const ButtonColor = {
  RED: 0,
  GREAN: 1,
  BLUE: 2,
} as const;
 
type ButtonColor = typeof ButtonColor[keyof typeof ButtonColor];
```

もう一つの理由としては、typescriptのenumはJavaScriptとして出力したときに複雑すぎることです。
最初にあげたenumの例をtypescriptコンパイラに通すと下記のような変数＋即時関数として
出力されます。

```ts
"use strict";
var ButtonColor;
(function (ButtonColor) {
    ButtonColor[ButtonColor["RED"] = 0] = "RED";
    ButtonColor[ButtonColor["GREEN"] = 1] = "GREEN";
    ButtonColor[ButtonColor["BLUE"] = 2] = "BLUE";
})(ButtonColor || (ButtonColor = {}));

var color = ButtonColor.RED;

console.log(color); // → 0
```

ではunion型でもas constでも大体できるのになぜ、enumを使うことがあるのでしょうか？

それはunion型や as constでは下記のようなbit演算を使って一つの値で複数の状態を表現できないことです。

下記はファイルのパーミッションをenumで表現した物です。

```ts
enum Permissions {
    None = 0,
    Read = 1,
    Write = 2,
    Execute = 4,
    Admin = Read | Write | Execute
}

// 読み取り、書き取り権限両方ら次のように表す。
let userPermissions: Permissions = Permissions.Read | Permissions.Write

// 書き取り権限を削除するなら下のように行う。
userPermissions = userPermissions & ~Permissions.Write

```

これをconst asで書いてみると下のようになります。

```ts
const Permissions = {
  None: 0,
  Read: 1 ,
  Write: 2,
  Execute: 4 ,
  // Admin: Read | Write | Executeとはオブジェクトリテラルでは書けない。
  Admin: 1 | 2 | 4,
} as const;

// 型定義をより柔軟にする
type PermissionKey = keyof typeof Permissions; // "None" | "Read" | "Write" | "Execute"
type PermissionValue = typeof PermissionsAsConst[keyof typeof Permissions];

// ファイルの読み書き、量権限
let userPermissions: PermissionValue = Permissions.Read | Permissions.Write;

userPermissions = userPermissions & ~Permissions.Write
```

あれ?やっぱりas constでもできるのではないか？と思う方もいるかもしれません。
ここで、vscodeなどのエディタを使っている場合、PermissionsValueにマウスカーソルを
移動させてください、ツールチップに型が表示されるはずです。
あなたの想像だと 0 | 1 | 2 | 4 | 7と表示されると思います。

しかし、PermissionValueはnumber型であると表示されます。これはなぜでしょうか？
これはas constとtypeの仕様として、オブジェクトリテラル内のプロパティに計算が含まれる場合は、その結果をコンパイル前に値を決定できないため、0 | 1 | 2 | 4 | 7と推論できないためです。
このため、enumとして使いたい型の値をリテラルと絞り込めないためです。
また、enumに新たな状態が増えるたびにAdminは1 | 2 | 4, Operratorは 1 | 2のように
どんどん具体的な数字が増えていき、数字がマジックナンバーと化します。
これはbit演算を多用するenumとしては非常に不便です。

このようなビット演算による状態の合成は、パーサーや設定ファイルのフラグ管理などの実装において頻出であり、as const では型の絞り込みが不完全になることから、enum を使うことが依然として実用的です。
実際、TypeScript のコンパイラ自身のコードベースにも enum は多用されています。これは、こうした用途において enum が依然として優れた選択肢であることを示しています。

なので、TypeScriptはenumは使ってはいけないというのは言い過ぎではないか？
というのが筆者の見解です。



## まとめ

長々と書いたけど
ライブラリのところの内容以外は大体必須。

Exclude, Extracts, Pick, Omitはライブラリ以外は、そんな使わない。

:::details ここまで読んでくれた人へ
最近記事を書くモチベーションが低下しています。
ここまで読んで「一定以上良いね」と思った人は❤️ボタン押してやってください。
励みになります!

読んでる人増えたら記事にもっと時間かけれるので、丁寧に突っ込んで書こうと思います。
:::

