---
title: "変数の値と代入について"
---

## 宣言と値の代入

powershellでは変数の宣言は必要無く、変数に値を代入するときは下記のように行います。

```powershell
$Name = "Hello world"
```

ほとんどのWebサイト上や教本では英語、日本語含めほとんどのサイトで上記のように値を設定していますが、
Powershellのスコープは特殊かつバグの温床になりやすい動きをしています。

そのため、特に理由がない限りは下記のように、
**その変数の初出時はSet-Variableを使って、変数のスコープと不変、または定数、値が書き変わうることを明示すること**を強く推奨します。


```powershell
# 定数として宣言
Set-Variable -Name abc -Scope local -Value "Hello World" -Option Constant

# これ以降は$abcという形で値を参照できる。

# Hello Worldと出力
echo $abc

# 値を再設定しようとするとここでエラーが表示される。
$abc = "Is not change variable"
```

下記のようにパイプラインの上流から受け取って、そのまま変数の設定をすることもできる。

```powershell
Invoke-WebRequest -Uri 'https://google.com' | Set-Variable -Name response -Option Constant
```

Powershellの定数にはConstantとReadonlyがありますが、
Powershellにおける、ConstantとReadOnlyの違いは
値の設定後に
Remove-Variable, Set-Variableに-Forceオプションをつけたときに値の削除や、変更ができるかどうかです。

下記の例を見てみましょう


```powershell
<#
.DESCRIPTION
Test Set Readonly Variable Work.

.PARAMETER Variable
Recieve string variable.


.EXAMPLE
Test-ReadOnly "Hello World"

.NOTES
This funciton is testing Set-Variable Commandled work.
#>
function Test-ReadOnly {
    param (
        [Parameter(Mandatory = $true)]
        [string] $Variable
       
    )
    Set-Variable -Name abc -Scope local -Value $Variable -Option ReadOnly

    Remove-Variable -Name abc -Force

}
```

```powershell
<#
.DESCRIPTION
Test Set Constant Variable Work.

.PARAMETER Variable
Recieve string variable.


.EXAMPLE
Test-Constant "Hello World"

.NOTES
This funciton is testing Set-Variable Commandled work.
#>
function Test-Constant {

    param (
        [Parameter(Mandatory = $true)]
        [string] $Variable
       
    )
    Set-Variable -Name abc -Scope local -Value $Variable -Option Constant

    Remove-Variable -Name abc -Force
}
```

この二つの関数をPowershell上で定義してみます。
下記のように実行すると、

```powershell
Test-Readonly "Test Variable!"
# 特にエラーなどは表示されない。
```

```powershell
Test-Constant "Test Variable"
# 下のようなエラーが表示される
# Remove-Variable: Cannot remove variable Name because it is constant or read-only. If the variable is read-only, try the operation again specifying the Force option.
```


対話形式でサクッと一度しか書かないならConstantやSet-Variableなど考えなくて

```powershell
$num = 123
```

のように初期化、して使って全く問題ないが、
スクリプトとして書き上げるときにバグの温床にはなる。

そのため、本書ではこれ以降は基本的にSet-Variableを使って変数の宣言と同時に値の代入を
行う。
