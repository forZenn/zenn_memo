---
title: "eval的なものの必要性とInvoke-Expressionについて"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["powershell", "メタプログラミング"]
published: true
---

powershellのeval。
eval的な処理はプログラミング入門書的なものでもさらりとしか書かれておらず、
入門書でちらっとこういうことできるよという知識だけ見て実際のソースコード
いまいちピンとこないと人も多いと思う。

こういうメタプログラミング的なものは、動的にソースコードを書きたいか、ソースコードの変更を少なくするために使う。

ライブラリをそれなりに書く人は使ったり、使うことを検討することはあるだろう。

例えば、下のソースコードを考えてみよう。

下はMavenのcentral repositoryへ検索apiを投げるロジックの一部抜粋だ。

```powershell
    ...
    # MaVen Central Repository Api
    $Uri = "https://search.maven.org/solrsearch/select"

    $queryParameters = @{
        q="$($query -join ' AND ')";
        wt="json";
        rows=100;
    }

    [BasicHtmlWebResponseObject] $response = $null
    if (5 -eq $PSVersion.Major -and 1 -ge $PSVersion.Minor) {
        $response =  Invoke-WebRequest -Body $queryParameters -Uri $Uri -UseBasicParsing
    } elseif (7 -ge $PSVersion.Major) {
        $response =  Invoke-WebRequest -Body $queryParameters -Uri $Uri
    } else {
        # $PSVersion 5.0 before version is not supported.
        Write-Error "Not supported Powershell Version."
        # throw Error
    }
  if ($response.statusCode -eq @(400, 404)) {
    Write-Error ""
     return
  }
   ...
```

#maven crentral repository のrest apiの仕様については下記を参考にすると良い。
 https://central.sonatype.org/search/rest-api-guide/

apiの使用を見ると+AND+と繋げろと書かれているが、Invoke-WeebRequestは半角スペースを+にUrlエンコーディングしてくれるため、
「 AND 」で条件を繋げた文字列をクエリパラメータに渡すとよい。

この処理は何をしているかというと、
Powelrshell 5.1なら-UseBasicParsingというswitchParameterを渡して、
Powershell 7以降なら-UseBasicParsingと使わずにそのまま実行するというコードだ。

なぜ、このようにPowershellのバージョンによって、パラメータを変える必要があるかというと、
Powershell5.1系はInvoke-WebRequestはhtmlパーサーとしてIEを使っているため、IEが無くなった今はIEではないParserを指定する必要があるため、UseBasicParsingという引数を渡す必要があるということだ。
ただし、core系のInvoke-WebRequestは最初からIEのパーサーを使っていないため、引数を渡す必要がないということ。
ではcore系も-UseBasicParsing渡せばええやんと思うかもしれないが、-UseBasicParsingはPowershell6.0以降は非推奨のパラメータになっており、新たにソースコードを書くときに使うべきではない。

これらの処理を分けるために、PowershellのバージョンごとにInvoke-WebRequestを書いているわけだが、これだと下記の問題が発生する。

1. UseBasicParsingが必要かどうかと判定ロジックとInvoke-WebRequestによる検索ロジックが制御文により密結合。
2. Invoke-WebRequestの戻り値を制御文の中で受け取る必要があるため、変数\$responseをimmutableにできない。

1も2もソースコードを長く管理することにおいて気になる点のため、できるだけ避けたいと思う。

1と2を同時に避けるためにはInvoke-WebRequestを動的に処理するということでしか回避できない。
単純にUseBasicParsingが必要かどうかの判定ロジックを他の関数に移動させても、結局if文を他の関数に追い出せないし、Invoke ~WebRequestが２回書くことに変わりは無いため、あまりソースコードの管理は楽にならない。

ということで、Powershellのevalに相当するInvoke-Expressiontを使うことになる。

## Invoke-Expression

Invoke-Expressionは文字列を渡すとそれをソースコード、スクリプトとして実行することができるコマンドレットだ。

Invoke-Expressionは下のような仕様がある。

1. 変数として渡された文字列を動的にパースして、スクリプトとして実行する
2. PowershellがInvoke-Exressionに文字列を渡すときに変数が含まれていた場合はその変数を展開してInvoke-Expressionに渡す。
3. Invoke-Expressionが動的にパースしてスクリプトとして実行するときに変数を展開、実行したいならInvoke-Expressionに渡すときに変数をあらかじめエスケープしておく必要がある。


文字で説明しても何のことかよくわからないと思うので、実際のソースコードで説明していきたい。

### 動作確認

簡単に動作を見ていく。
ここではInvoke-Expressionに実際に変数を渡してみてそこで動作の違いを見ていく。

### 変数を渡さない場合

一番オーソドックスというか変数も何も渡さずに文字列だけ渡してみる。
これだとInvoke-Expressionを使う必要は特に無いが例として理解して欲しい。

```powershell
# Get-Help Invoke-Expressionと展開されて実行する。
Invoke-Expression "Get-Help Invoke-Expression"
```

なんて事はないがInvoke-Expressioonは受け取った文字列を動的にスクリプトとして
実行するコマンドレットであるということだけ覚えててほしい。

### 変数が文字列の場合

ここからは実際に変数を使ってみる。

Invoke-Expressionは変数を渡した場合は変数をエスケープする
かそうでないかで実際の動作が変わってしまう。

下のソースコードは

まずは変数をそのまま渡してみよう

```powershell
$fileName =  test.txt
#=> New-Item -Type File test.txtが動的に実行される。
Invoke-Expression "New-Item -Type File $fileName"
```

この場合はNew-Item -Type File test.txtという文字列をInvoke-Expressionが受け取って、
動的にNew-Item -Type File test.txtを実行しているということになる。

その次は変数をエスケープして渡してみよう。

```powershell

# バックスペースで$をエスケープして変数名を文字列で渡す。
#=> New-Item -Type File $fileNameが動的に実行される。
Invoke-Expression "New-Item -Type File `$fileName"
```

この場合はNew-Item -Type File \$fileNameという文字列をInvoke-Expressionが受け取って、
動的にNew-Item -Type File \$fileNameを実行しているということになる。

上二つの例の場合はエスケープしてもしなくても\$fileNameの変数には文字列が入っていて、
test.txtファイルが作成されるという結果が変わらないため、特に問題は生じない。

eval系で問題になるのはオブジェクトを渡すときだ。
文字列に埋め込んで変数として展開するのと、直接変数を渡すので全く値が変わる。

以降の文章にある配列の場合と連想配列の場合を見てほしい。

### 変数が配列の場合

ここでは下記のようにGet-Process | Select-Objectとして、Powershellで取得した、
Processのオブジェクトの指定のプロパティを表示したい。

```powershell
$Property = @("ProcessName", "Id", "WS")

Get-Process | Select-Object -Property $Property
```


```powershell
$Property = @("ProcessName", "Id", "WS")
Write-Output "${property}"
#=> ProcessName ID WS

Get-Process | Select-Object -Property $Property

Get-Process | Select-Object -Property ProcessName, Id, WS

# 
# Get-Process | Select-Object -Property ProcessName ID WS 
Invoke-Expression "Get-Process | Select-Object -Property $Property"

# 正しくは下のように実行する。下記のコメントにパースされて動的に実行される。
# Get-Process | Select-Object -Property $Property
Invoke-Expression "Get-Process | Select-Object -Property `$Property"
```

### 変数が連想配列の場合

次に変数が連想配列の例をあげていく。
ここではInvoke-WebRequestを使ってみようl


```powershell
    $Uri = "https://search.maven.org/solrsearch/select"
    $queryParameters = @{
        q="p:maven-archetype AND g:org.apache.maven.archetypes";
        wt="json";
        rows=100;
    }

# ここではプロジェクトのスタータ-に使われるarchetypesのプロジェクトで、mavenにあるものをリストで出してみよう。
# https://search.maven.org/solrsearch/select?q=p:maven-archetype+AND+g:org.apache.maven.archetypes&wt=json&rows=100
Invoke-WebRequest -Uri $Uri -body $queryParameters

# 下のコードを実行するとコメントに書かれているソースコードにパースされて、連想配列の文字列表現である「System.Collections.HashTable」という文字列を渡すため、エラーになる。
# Invoke-WebRequest -Uri https://search.maven.org/solrsearch/select -Body System.Collections.Hashtable
Invoke-Expression "Invoke-WebRequest -Uri $Uri -body $queryParameters"

# 正しく実行したいなら$をエスケープする。コメントに書かれているコードに動的にパースされて実行される。
# Invoke-WebRequest -Uri https://search.maven.org/solrsearch/select -Body $queryParameters
Invoke-Expression "Invoke-WebRequest -Uri $Uri -body `$queryParameters"
```

文字列と配列の展開のされ方で何となくわかったと思うが、連想配列の場合もエスケープする必要がある。


## 問題のソースコードの修正

xmlなどのオブジェクトも同様にオブジェクトの名前を渡す。

ということで、 *数字と文字列以外を渡す場合はエスケープが必須*と考えると良い。

以上を踏まえて上のInvoke-WebRequestのソースコードは下のようになる。

```powershell
    $UseBasicParsing = ""
    if ($PSVersion.Major -eq 5 -and $PSVersion.Minor -ge 1) {
        $UseBasicParsing = "-UseBasicParsing"
    } elseif (7 -ge $PSVersion.Major) {
        # Default parsing

    } else {
        # $PSVersion 5.0 before version is not supported.
        Write-Error 
        # throw Error
    }
    #=> Powershell 5.1系列の時 Invoke-WebRequest -Body $queryParameters -Uri "" -UseBasicParsing
    #=> Powershell 7以降のとき Invoke-WebRequest -Body $queryParameters -Uri "" 
    $response = Invoke-Expression "Invoke-WebRequest -Body `$queryParameters -Uri $Uri $UseBasicParsing"
```

結構ええ感じに処理が吸収されたと思う。

## デメリット

1. eval的な処理はインジェクションに脆い。特にプログラミング言語やシェルスクリプトのインジェクションは安全にエスケープする方法がない(気になる人はosコマンドインジェクションでググる)。
2. 言語のパーサーを信用することになる。
3. 速度が遅い
4. いつのタイミングで文字列に展開されるかがややこしい。
5. 動的に実行するため、エディタがこのコードおかしいよ！と赤文字で言ってくれない。
6. エスケープしてeval系の処理に渡しているため、linterに変数が使われていないと怒られる。

1は他人からの入力をInvoke-Expressionに渡さないということで回避できる。
2はパースに癖やバグに近い挙動がある言語(VBAとかcmdとか)では使わないということにするしかない。
3はあきらめましょう。トレードオフです。
4,5は慣れです。気をつけないとややこし過ぎて禿げます。
6はlinterに定義されているignore的なコメントを書くことで回避できる。

「うちのプロジェクトだとデメリットの方が多いじゃん！」と感じるなら使うのをやめること。
そう感じることも多いから、使わないこと多いのよ。

ライブラリ作る人以外要らん機能な気もする。

でも知ってたら得する時もある微妙な存在。

## まとめ

PowershellでInvoke-Expression使おうと思ったら一度これ見て考えてください。

メタプログラミングはかなりプログラミングスキルの差が出るから、書くときよく考えてね。
他人に説明するのメンドイなら使わないのも手。
でも、変更コストとソースコードの再利用性考えるとメタプログラミング強要されることそこそこあるのよね...

技術力とコミュニケーションコストをどう見積もるか？ですな。
少人数のプロジェクトなら、割と問題にならんのですが、使いたくなるプロジェクトに限って大抵規模デカいのがね...
