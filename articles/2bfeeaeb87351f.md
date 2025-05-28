---
title: "なぜこのtsの機能を使うのですか？"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["contest2025ts", "ts"]
published: false
---

と聞かれたときにちゃんと説明するための記事です。
よく聞かれるので、聞かれたときように書きました。

tsの本質はドキュメンテーションとlinterです。
tsとjsは割と独特過ぎるので、他の言語から来た人だと
混乱するところも多い。

でも入門書を見ると、初心者ならこれ何に使うねん？！とツッコミ入れたくなる
ものもあると思うので、それについて書いていきます。

例としてdenoを使っていますが、
main関数の宣言方法が違うだけでそれ以外は
nodejsでも差はないです。

特に断りが無い限り一緒に書いているjsdocとコード上、linter上全く同じように動作します。


### 型定義

```ts

type hello = string

```

jsdocでの表記

```ts
/** @typedef {string} hello */
```

他の言語から来た人はパッと見、
ドキュメンテーションなのか、コードなのかよく分からなくなので、
気持ち悪いと思う人も多いと思う。
筆者も正直気持ち悪くてこれ嫌いです。

**慣れましょう**

typeと言っているが、実際は他の言語でいう、aliasに近い動きをしている。



### interface

至る所で出る。
使ってないライブラリも現場のコードもない。
typeとinterfaceの使い分けだが、
オブジェクトの定義はこれで、型の定義はtypeと考えると良い。

1. classのインスタンス云々の意味のオブジェクト
2. オブジェクトリテラルの定義

他の言語から来るとinterfaceが異様に見えるが、
**慣れましょう**

```ts

```

jsdocでの表記

```ts

/**
 * @interface {Object} Point
 * @propterty {number} x
 * /
```

### typescriptでは

値自体も型として扱うということだ。

### 固定長の配列

配列というか、タプルとしてよく使う。

```ts
// 各々のインデックスに
type aaaa = [number, number, number];

const [1, 2, 3];

```

みんなお馴染みreactのState関数に使われている。
この機能を使って、
useState()は
gettterと変数とsetterを提供している。

```ts

const [count, setCount] = useState<number>(0);
```

```js

/** @type {[number, React.Dispatch<React.SetStateAction<number>>]} */
const [count, setCount] = useState(0);


// 初期値がない場合、booleanとundefined両方がありうるため、下のようになる。
/** @type {[boolean | undefined, React.Dispatch<React.SetStateAction<boolean | undefined>>]} */
const [] = useState();
```

### discriminated union\(判別可能なUnion\)

判別可能なUnionという訳がいろんなところにあるが、
テクニック名の和訳かつ、
Unionが判別するのでなくて、Unionで状態を判別するのだから、
Union判別型、Union判別法が自然な訳な気がする。
特殊な構文や機能が新たにあるわけじゃないし。

Unionに制御構造持たせようとすると、自然とこの書き方に行き着く。


アプリケーションコードで頻出。



たまにstatusのフラグを1, 2, 3, というふうに
している人を見かけるが、非常に危険なので止めること。

多くの場合、
後で条件が増えていって、
1, 2, 3がマジックナンバーと化す。

なんで、こんな危険な書き方している人が多いのか不思議だったが、
サバイバルtypescriptに数字の例が書いてあった...

https://typescriptbook.jp/reference/values-types-variables/discriminated-union

数字で書く場合はhttpsのstatusコードとか、もうそれ自体が
実質的に判別型Unionとして機能していて、
今後変更されないことが確定しているみんなが使うもの以外は良くないかなぁ...

例えば、次のような会員システムを考えてもらいたい。

黎明期はこの書き方しないと怒られたのかもだが、今は１バイトが貴重な時代じゃないからなぁ...
保守のために蹴ることはないはず。

後からその会員は２種類3種類にふえませんか？
無効な会員も退会した人間と休眠状態の会員と、
無料会員を1有料会員を2,無効な会員を3とした後に
Vip会員が増えませんか？それを4に割り当てたりしませんか？
ひどい場合は数字であることを利用して、
無効な会員を3以上として、

会員を判定するのに===3のように判定しており、
それに合わせて一部のスタイルを変更していたりしませんか？
無効な会委員が



### 制御文

ifやelse, switchによる制御文により、
自動的にその時点での型が決定される。


### 型とinterfaceのimport export

#### type

export, importにtypeをつけます。

```ts
export {

}
```

```ts
import type {} from '';
```

#### interface

interfaceはts上ではjsのオブジェクトと全く同じ形で
import, exportできる。


### Union型

めちゃくちゃ使う。

型のまたは\(||\)と覚えよう。

typescriptは**単なる値も型としてもみなす**ため、
Union型は下のようにenumの代わりとして使うことが多い。

```ts

type os = "Windows" | "MacOS" | "Linux";

// エラーが出る。 Type '"BSD"' is not assignable to type 'os'
const yourOS: os = "BSD";

```

nullableな型は下のようにする

```ts
type os 
```

### as \(cast\)

typescriptでは他の言語と同様にcastを使って、
強引に変数に型を割り当てることができる。


```ts

// 状況的にこの型以外ありえないため、cast。

let 
```

### バリデーションライブラリ

typescriptの文法や機能ではないが、
型の説明をするのに必要なので、ここに書く。

Zodやjoiなど、
主にフォームなどの外から来た値のバリデーションに使う。

これらがないと、
1. プログラマは永遠とasや型ガードを使う必要がでたり、
2. 有効なメールアドレスか？など型ガードとして実装するのが難しい処理を

書いていく必要があり、大変過ぎる。


ここでZodのソースコードを見てみよう。

型ガードや、asを書く量を減らすためのライブラリと
思ったらええかと。

### インデックスシグネチャ

javascriptの場合は動的にオブジェクトリテラルにキー、プロパティを追加できる。

```js
const point = {
    x: 30,
    y: 50,
}

// 元々無いキーを同的に追加できる。
point.z = 100;
```

これはtsでは禁止されており、明確にコンパイルエラーになる。
最近ではjsでも、エディタ上では黄色い下線で警告が出る。

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

point.uu = "Unsigned Value";
```

動的にキーを追加できるのが普通になると
typescriptの意味がなくなると思うかもだが、
厳密に型定義してる意味がなくなる。

アプリケーションコードでは来る値が決まっているため、
下のように

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

reactのコンポーネント関係のライブラリによく使われる。
コンポーネントに動的にキーを追加したいとか。

jsからtsに移行したときに動的にプロパティを追加している部分が多くて、
工数的に難しいときにとりあえず、インデクスシグネチャを使って、
tsにしていることは多々ある。
js時代からts時代までお金を埋めているシステムだし、
確実に長期運用が見込めるため、
無理にts化する必要はないとは思うが。
後述するesm+jsdocで解決できないか負担の比較を考えること。




ここでCreativ Timのソースコードを見てみよう。

河川

例１ システムの言語

例えば、システムが複数の言語設定とそれに付随する

```ts

type Keyboard = "jis" | "apple_jis" | "us";

// プロパティ分けていくときりないので、inputMethodはObjectにしているが、
// もちろん特定のInputMethodのオブジェクトが入る。
interface LanguageDetails {
    inputMethod: Object;
    keyboard: Keyboard;
}

interface Language {
    [language: string]: LanguageDetails;
}
```

インデックスシグネチャもjsdocで全く同じように表現できる。

```ts

/**
 * @typedef {"jis" | "apple_jis" | "us"} Keyboard
 * 
 * @interface LanguageDetails 
 * @property {Object} 
 * 
 * @interface Launguage
 * @property {[lauguage: string]}
 * /
```


### ジェネリクス

ジェネリクスとは型を引数として取れる、関数、オブジェクトだ。


次のように

```ts
```

でも、アプリケーションコードの場合は大抵、ジェネリクスにも入ってくる型が数通りで決まるため、
ジェネリクス型制約を使うことで、型を縛ること。

型制約を使わずにジェネリクスを使うことはライブラリでは多々あるが、
アプリケーションコードでは少ない。

```ts
<T extends string | number>
```

他のプログラミング言語だと使ったことはあっても自分で実装したこと無い人も多いと思う。
実装しても単純なものを指定通りに実装するとかそれ系。
ただ、tsだとやたら使うことになるので、
ここは例を挙げたい。

uiだとコンボボックスやMenuItemなど
リスト周りの処理で使う。

ここは確実に後で、コンボボックスとして見せたい、MenuItemとして見せたい
などのいろいろなニーズがあるが、それを書いてる最中に求められないなど。

ラジオボタンか？Selectボックス,CheckBoxか？
というのをギリギリまで決める必要がある時がある。
差し替えが非常に楽になる。

```ts
import Radio from '@mui/material/Radio';
import RadioGroup from '@mui/material/RadioGroup';
import FormControlLabel from '@mui/material/FormControlLabel';
import FormControl from '@mui/material/FormControl';
import FormLabel from '@mui/material/FormLabel';
```
https://mui.com/material-ui/react-radio-button/

https://mui.com/material-ui/react-checkbox/

大抵の場合は継承で済むので、より汎用性が求められたら使う。
ライブラリかな。


### readonly

classのプロパティ、interfaceのプロパティに使える。

interfaceに使えるということはオブジェクトリテラルのプロパティを
固定化できるということ。

1. クラス定義時の初期化
2. コンストラクタでの初期化、

csharpっぽい感じになる。


```ts
```

### 列挙型

使ったらアカんやつ。tsの黒歴史。

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

typeで型パズルしまくってるとtypescriptからesm+jsdocへの移行は難しい,
jsdocで表現できないtypescriptのコードがある。
などの問題はある。

型のimport

```ts
// import('').DefinedTypeで型を指定してもWebStormだとツールチップ上で正しく表示されないっぽい。　

/**
 * @type {import('../path/to/library').DefinedType} 
 * /
```

## ライブラリ用の機能

型パズルする系が多い。
普通にシステム組むだけなら無くても困らない。

型パズル用の機能。
~~ライブラリでもないのに使うやつのコードはたいてい読みづらい~~
変更に強くなるとか、運用楽になるなら使ってもええんじゃない？
覚える優先度は低く、知らなくてもコードでもドキュメンテーションでも代替可能。

型を集合のように扱っているのが直感に反する。

#### Exclude

typescriptは型を集合の要素のように扱うことができる。

型のxorと覚えよう。

ライブラリ以外基本的に使わない。

型パズルみが出てくるので、

#### Extract

型パズル。ライブラリ以外では使わない。
型の
正直使わない方がいいと思う。
...Containsの方が名前の方が妥当な気がする。
Genericにオブジェクトリテラルを渡すことになるのが結構気持ち悪い。


### Pick

typescriptはオブジェクトのプロパティを集合の要素のように扱うことができる。

動的に型にオブジェクトにプロパティを追加した型を作成する。


### Omit

動的に型にプロパティを削除する。

## まとめ

長々と書いたけど
アプリケーションコードを書くだけなら、
interface, type, union, discriminated Union, ジェネリクス、インデックスシグネチャ、型ガードさえ抑えとけばあとはどうにかなる。
Exclude, Extracts, Pick, Omitはそんな使わない。

generics 