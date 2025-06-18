---
title: "TypeScriptの強力な機能たち：なぜそれらを使うのか？"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["contest2025ts", "ts", "typescript"]
published: false
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

// CountDown型はnumber型と推論される。　
type CountDown = typeof countDown[number];


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

「typescriptは型があるから、曖昧じゃなくて安全！」とか
よく言うが、もっと具体的に言えとか、
下手したらもっと意味が無くて個人的なか着心地云々の

キーを追加できることである。
javascriptの一番のバグの温床になっていたのは
型がないことでは無く、
キーを動的に追加することができるのに、
これと判別可能なユニオンを作る習慣も無いために
下手くそは無限に下手くそに描ける作りになっていたからだ。



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

####　ジェネリクス型制約

しかし、アプリケーションコードの場合、ほとんどのケースでジェネリクスとに入る方は数通りに限定
されます。そのため、ジェネリクス型制約を使うことで、型をより厳密に縛ることが推奨されます。

型制約を使わずにジェネリクスを使うことはライブラリでは多々ありますが、アプリケーションコードでは
その頻度は少なくなります。

```ts
// ジェネリクス型制約は <T extends 型1 | 型2>という表記になる。
// Tはstringまたはnumberのいずれかの型に限定されます
function printId<T extends string | number>(id: T): void {
    console.log(id);
}

printId("abc"); // OK
printId(123); // OK
// printId(true); // エラー: 型 'boolean' を型 'string | number' に割り当てることはできません。
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
    y: 40,
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

残念ながら**readonlyはJSDocではエラーになりません**ので、
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

しかし、これを行うと、OS型はstring型であると推論されてしまいます。


```ts

const osNames = ["MacOS", "Windows", "Linux"] as const;

// osNamesの各要素がリテラル型として推論され、それをUnion型として結合します。
type OS = typeof osNames[number];
```

上のコードは下記のようなUnion型の型宣言と同じ意味になります。

```ts
type OS = "MacOS" | "Windows" | "Linux";
```

これぐらいだと全く有り難みがわからないが、例えば
オブジェクトの場合を考えてみよう。

styleとかのライブラリ見るとよく見かける。

```ts
const invoiceMessage = {
    success: "請求書を作成しました。",
    info: "請求メッセージです。",
    error: "請求書の作成に失敗しました。"
} as const;

type os = typeof MacOS.distributed
```

zodのようなバリデーションライブラリを使う時でも併せてよく使います。

### Enum

使ったらアカんやつ。tsの黒歴史。
Enumという言葉を見かけたら普通にプログラムで使うやつだ！
と思って使い出すと思います。それが本来

```ts
enum ButtonStatus {
  ON,
  Monkey,
  Lion,
  Bear,
}; 

```

typescriptのtsconfig.tsの実装の至る所で使われている。

typescriptでは
下記のようにUnionやinterfaceで解決できるため必要性がない。
使わんでください。
jsでも無くても困らない機能。

```ts

type os = "MacOS" | "Windows" | "Linux"

interface System {
    os: "MacOS" | "Windows" | "Linux"
}
```

tsをcsharpに寄せようとしてこの機能追加したんかな？

:::details ライブラリの例

githubの[microsoft/Typescript](https://github.com/microsoft/TypeScript/tree/v5.8.3)を見ればわかるが、
src/compiler/配下の割とコアな部分に満遍なく使われている。

<!-- 特に言語周りのパーサー部分はenumで実装する部分が多いので、
enum使ったほうがパーサーの実装が楽だったりしたのかもしれない(知らんけど)。
ちなみにPowershellも割とパーサーはenumで結構使ってる。(global, script, functionなどのスコープはenum)

src/compiler/checker.tsを読むのは大変だが、
enumを頼りに読むとそこそこ読める。
tsconfig.jsonの

```typescript
const _computedOptions = createComputedCompilerOptions({
    ...
```

元々はtypescriptのcompilerののための機能だったりしたのかも? -->

:::

### unknown型

anyをより安全にした感じの方。
インスタンスのnullableがより安全な型のように、

anyはプロパティにアクセスしても、実行時までエラーか分からないし、
コンパイル時にエラーとして処理されないが、

unknownはエラーとして処理される。
型ガードによって、型を絞りこむ、castするなどの

```ts

```

### never型

型というより、変数がそのブロックに行かないことを示す、識別子、マーカー。

必ず辿り着かないときにその型になる。

voidは正常終了する関数と覚えたらいい。

ライブラリで使われるというより、なんか気がついたら出てるので
その程度の知識で良い。
vscodeでもwebstormでもtooltipや下線でエラーが表示されるので、
それを見て対処すれば困ることはないだろう。

先ほどのOS型を使って考えてみよう。
osごとによってキッティング作業が違うとする。

```ts
type OS = "Windows" | "MacOS" | "Linux";

function kitting(os: OS) {

}

if (OS)

```

switch分でも同様に

### 型ガード

typescriptではunion型を扱うことができるが、
どの時点でどの型が入っているのか分からないことが起きる。
また、js、tsでは頻繁にundefinedが入ってくるので、それを弾くために
行う。　

これはif文などの制御文を使うことにより、
論理演算によりそのブロックでは存在しない型はなくなるその時点での
型を絞り込むことができる。

例えば、早期リターン

例１

```ts
function getOSTimeZone() {
    if ( === "macOS" ||  === "Linux") {
     
     return {}
    }

    // ここから先は === Windowsとして判断され、
    // エディタのtooltip上でも

}
```

例２ 製品の価格

商品によってはサイト上で、
2000円、3000円じゃなくて時価や価格未定などの
文字列も入れたい時があるとする。

その状態によって、ページの別の場所に
時価、未定に関する注意書きが書かれるようになるケースを考えよう。

```ts
type price = number | "時価" | "未定";

function 
```

### 型ガード関数

discriminated Unionと合わせて読む。
アプリケーションコードでも結構使う。

上で論理演算と制御文によって、typescriptが自動的に
ブロック上での型を絞り込むことが分かったと思う。

しかし、ここで次のような条件を考えてみよう。

型ガード関数は下の条件を満たすと型ガード関数として身される。

1. true or falseをreturnする
2. 戻り値に\(型判定をしたい引数 is 型\)とする

型ガード関数はtrueさえ返せば、

```ts

type MacOS = "Macですよ。";

function isMac(name: unknown, _: boolean): name is MacOS {
    return true;
}

const os:unknown = undefined;

if (isMac(os, false)) {
    // このブロックで内では変数osは
    // macOS型とtypescriptトランスパイルが解釈し、
    // エディタのツールチップ上でもMacOS型と表示される。
    os
}
```

残念ながら型ガード関数はasync付きの非同期関数として定義できないので、

型ガードは下のように非同期関数として定義できないということだ。

```ts

async function getOs(os: OS): os is "MacOS" {
    // let os: OS;
    try {
        await $`command -V sw_vers`;
        return true;
    } catch {
        return false;
    }
    
}
```

```ts
function isMacOS(os: OS): os is "MacOS" {

}
```

```ts
function 
```

関数まで見て逝去文

有名なものだとdenoの標準ライブラリに使われている

型ガード、型ガード関数は全てasで簡潔に書くことができるが、

## おまけ

### esm + jsdoc

構文というより設計の話。
typescriptの本質ドキュメンテーションとlinterなので、
それをjsdocで行おうということ。

ただのjsのためtypescriptより保守性に優れる。
長期間運用するシステムorライブラリにむく。

有名なライブラリだとsolidusがtypescriptからesm+jsdocに移行した。
denoのcore部分がtypescriptからems+jsdocに移行した。
技術的にどうこうというより、感情的な反発が大きいから
組織でやるならワンマンか言語に拘りが薄くないと無理だと思う。

型パズルしまくってるとtypescriptからesm+jsdocへの移行は難しい,

移行の際に

1. 一部、jsdocで表現できないtypescriptのコードがある。
2. エディタの保管がtypescriptとほど賢くない。

などの問題はある。
エディタの補完はvscodeが今のところ一番賢いかな？触って見た感じ。msのツール同士だから当然と言えば、当然な気もするが。

感じ方の問題ではあるが、
jsdocからtypescriptの機能を使っているから、
typescriptであると主張するのは正直厳しいとは思う。
typescriptのコードを殆ど無くして、jsdocを書くのがメインになったら、
それはtypescriptを使わなくなったというのでは？
pythonやnodejsからlibssh使ってて、本質的にはcを使っている！みたいな話ですし。
そりゃ本質的にはc使ってるけど...

#### 型のimport

```ts
// import('').DefinedTypeで型を指定してもWebStormだとツールチップ上で正しく表示されないっぽい。　

/**
 * @type {import('../path/to/library').DefinedType} 
 * /
```

### ライブラリ用の機能

型パズルする系が多い。
普通にシステム組むだけなら無くても困らない。

型パズル用の機能。
変更に強くなるとか、運用楽になるなら使ってもええんじゃない？
覚える優先度は低く、知らなくてもコードでもドキュメンテーションでも代替可能。

下記に挙げた機能は、
型を集合のように扱っているのが直感に反すると感じる人はいるかもしれない。

#### Exclude

typescriptは型を集合の要素のように扱うことができる。

型の部分集合と覚えよう。

ライブラリ以外基本的に使わない。

型パズルみが出てくるので、

#### Extract

型パズル。ライブラリ以外では使わない。
型の
正直使わない方がいいと思う。
...Containsの方が名前の方が妥当な気がする。
型パズルやってない人からすると、
Genericにオブジェクトリテラルを渡すことになるのが結構気持ち悪い。


#### Pick

typescriptはオブジェクトのプロパティを集合の要素のように扱うことができる。

動的に型にオブジェクトにプロパティを追加した型を作成する。


#### Omit

動的に型にプロパティを削除する。

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

