---
title: "Powershellからrustを呼び出す"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["powershell", "rust", "windows", "linux", "mac"]
published: true
---

Powershellからrustのソースコードを呼び出す方法が英語でも日本語でも無かったので、ここに書く。

msはかなり意欲的にrustに取り組んでいるし、今後rustからwindowsのapiを叩けるライブラリも増えるはずなので、今後rustにwindowsの資産は増えていくだろう。

ここでは書き方と構成、github actionを使ったPSGalleryへの登録までの処理含めて一気通貫に説明する。

## rustと連携するメリット

低レイヤーを操作することと、かつcpp並みの速度、メモリ効率を得るためです。

Powershellは作り上、速度、メモリ的に本当にどうしようもないので、これらについての処理は他の言語に任せることになります。

バックアップアップスクリプトやバイナリ操作、DB操作、バッチ処理などに使うとよいと思います。

## 対応するアーチテクトについて


今回は対応するアーチテクとPowershell 5.1系列、Powershell core系の7.2(dotnet 6.0), 7.3, windows, mac, linuxの
x86_64で動作するrustを使ったPowershellのスクリプトと、モジュールを書く。

これは現時点でgithub actionに対応しているアーチテクと全てである。

armについても同じ容量でできるが、ローカルでビルドすることになる。

ただ、github actionもじきarmの対応はするだろう。

また、Powershellのモジュールの形式として、バイナリモジュールではなく、スクリプトモジュール形式を使う。
これは5.1系列とcore系列両方を対処しながら、低レイヤーで主要なアーキテクトで動くように設計する場合、バイナリモジュールだと運用の負荷が大き過ぎるためである。

お金もらえるならやり方書きます

## 他の言語との比較

同じくwindows対応めっちゃ頑張っている言語としてgolangがある。
powershellにおいてgolangとrustとの比較についてはrustはmsvcを使っているのに対してgolangはgccを使っている点。
golangの方がwindowsに対しては枯れている点。
速度はrustの方が早いが、デバッグは明らかにgoが得意で、ffiの言語としては低レイヤーが得意なgoもかなり優秀。
dotnetのネイティブコンパイルは最近ようやくプレビューとして実装されたため、５年は見送った方が良さそう。
swiftもできるけど、
というのが差別化ポイントか。

## サンプル

サンプルは[github](https://github.com/KatsutoshiOtogawa/PowershellInvokeRust)か、
[PSGallery](https://www.powershellgallery.com/packages/PowershellInvokeRust/0.0.10)においたので見てください。

## 実装するPowershellの関数

今回は最も単純なMeasure-Add functionを実装してみる。

これは基本的なx + yの結果を返す関数で、Powershellの命名規則に従って、Measureという接頭辞を追加したものだ。

## 構成

powershellからcsharpのソースコードを呼び出して、そこからrustでビルドされた共有ライブラリを呼び出す。

Powershellから直接共有ライブラリを呼び出せないので、一度csharpのソースコードをかますことになる。

## rustのソースコード

特に難しい点は無い。
最も基本的なffiになる。

```rust:rust_src/src/lib.rs
#[no_mangle]
pub extern "C" fn add(x: i32, y: i32) -> i32 {

    return x + y;
}
```

tomlファイルは下記のように書く。
通常と違うのは\[lib\]という項目を増やして、
crate-typeにcdylibを追加することだ。
こうするとビルド時に共有ライブラリを作成してくれる。

```toml:rust_src/src/Cargo.toml
[package]
name = "rust_src"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[lib]
crate-type = ["cdylib"]

[dependencies]
```

ビルドすると

```
cargo build --release
```

target/architecture名/release/{suffix}{package.name}.{ext}

というふうに共有ライブラリが作成される。

suffixの部分はlinux, macはをlibに、windowsは空文字列になります。
package.nameはCargo.tomlの\[package\]のnameという項目が使われます。
extの部分はwindowsはdll, macはdylib, linuxはsoになります。

## chsarpのソースコード

ひねる必要がある。
ここは先ほど作成したrustの共有ライブラリを実行するエントリーポイントになります。

このレイヤーではpowershellに対応したバージョンのdotnetのdllバイナリを
作ることが目的です。
dotnetのバージョンとアーキテクトに対応している関数を提供し、それをPowershell
から実行することにより、アーキテクトごとに正しい処理を呼び出します。

dotnet 6.0以上と、.netframework4.8で全然作りが違うので、
プリプロセッサで処理を分けることと、
DllImportに実行時に探すビルド済みのdotnetのdllからの相対および絶対パスを指定します。

ここではrustに送る処理もpowershellから呼び出す処理も同じ関数に
マッピングしてますが、csharpで処理してからrustに送りたい処理があるなら、
rustとマッピングしている関数とpowershellから呼び出す関数と分けましょう。

```cs:csharp_src/lib/Math.cs
using System;
using System.Runtime.InteropServices;

public enum SystemArch {
    Unknown,
    X64,
    Aarch64
}

namespace lib {

#if NET6_0_OR_GREATER
    public class Math
    {

        // このcsharpのdllからの相対パスか絶対パスを書く
        [DllImport("../../share_lib/x86_64-apple-darwin/librust_src.dylib", EntryPoint = "add")]
        public static extern Int32 add_x86_64_apple_darwin(Int32 x, Int32 y);

        [DllImport("../../share_lib/aarch64-apple-darwin/librust_src.dylib", EntryPoint = "add")]
        public static extern Int32 add_aarch64_apple_darwin(Int32 x, Int32 y);

        [DllImport("../../share_lib/x86_64-unknown-linux-gnu/librust_src.so", EntryPoint = "add")]
        public static extern Int32 add_x86_64_unknown_linux_gnu(Int32 x, Int32 y);

        [DllImport("../../share_lib/aarch64-unknown-linux-gnu/librust_src.so", EntryPoint = "add")]
        public static extern Int32 add_aarch64_unknown_linux_gnu(Int32 x, Int32 y);

        [DllImport("../../share_lib/x86_64-pc-windows-msvc/rust_src.dll", EntryPoint = "add")]
        public static extern Int32 add_x86_64_pc_windows_msvc(Int32 x, Int32 y);

        [DllImport("../../share_lib/aarch64-pc-windows-msvc/rust_src.dll", EntryPoint = "add")]
        public static extern Int32 add_aarch64_pc_windows_msvc(Int32 x, Int32 y);
        
    }
#elif net48
        [DllImport("../../share_lib/x86_64-pc-windows-msvc/rust_src.dll", EntryPoint = "add")]
        public static extern Int32 add_x86_64_pc_windows_msvc(Int32 x, Int32 y);

        [DllImport("../../share_lib/aarch64-pc-windows-msvc/rust_src.dll", EntryPoint = "add")]
        public static extern Int32 add_aarch64_pc_windows_msvc(Int32 x, Int32 y);
#endif
}
```

csprojは下のようになります。

TargetFrameworksに使いたいPowershellの
バージョンとコンパチブルなdotnetのバージョンを書いてください。
この設定を見て必要なdotnetバージョンと対応したdllファイルを作成します。

今回はPowershell7.3(net7.0), Powershell7.2LTS(net6.0),
Powershell5.1(net48)の3つを指定しました。

```xml:csharp_src/lib/lib.csproj
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net7.0;net6.0;net48</TargetFrameworks>
    <ImplicitUsings>disable</ImplicitUsings>
    <Nullable>disable</Nullable>
    <Authors>OtogawaKatsutoshi</Authors>
    <Copyright>Copyright (c) 2022 OtogawaKatsutoshi</Copyright>
  </PropertyGroup>

	<PropertyGroup Condition="'$(TargetFramework)'=='net6.0' OR '$(TargetFramework)'=='net7.0'">
	  <Nullable>enable</Nullable>
	</PropertyGroup>
</Project>

```

特にmsbuildが云々など気にしなくてもnet48はビルドされますが、
dotnetのビルド環境として、.netframeworkも含む場合は
windowsをビルドに使ってください。そっちの方がトラブルの検査も簡単です。
ビルドはソリューションファイル(sln)と同じ場所で下記のようにコマンドを実行するとビルドされます。

```pwsh
dotnet build --configuration Release
```

こうすると、bin/Release/配下にnet6.0, net7.0, net48とtagetframeworksに対応した
ディレクトリが作成されます。
この各ディレクトリにある.dllファイルをPowershellから使うことになります。

余談としては今回は行いませんが、下のようにcsprojにdefineプリプロセッサを定義することで、
ビルドした環境がwindows, mac, linux, cpuごとに処理を分けることができます。

```xml
  <PropertyGroup>
    <DefineConstants>$(DefineConstants);CORECLR</DefineConstants>
    <IsWindows Condition="'$(IsWindows)' =='true' or ( '$(IsWindows)' == '' and '$(OS)' == 'Windows_NT')">true</IsWindows>
  </PropertyGroup>

  <!-- Define non-windows, all configuration properties -->
  <PropertyGroup Condition=" '$(IsWindows)' != 'true' ">
    <DefineConstants>$(DefineConstants);UNIX</DefineConstants>
  </PropertyGroup>

  <!-- それぞれのOSアーキテクチャ向けにビルドされたか？ -->
  <PropertyGroup Condition="'$(RuntimeIdentifier)' == 'linux-arm64'">
    <DefineConstants>$(DefineConstants);LINUX_ARM64</DefineConstants>
  </PropertyGroup>
  <PropertyGroup Condition="'$(RuntimeIdentifier)' == 'linux-x64'">
    <DefineConstants>$(DefineConstants);LINUX_AMD64</DefineConstants>
  </PropertyGroup>
  <PropertyGroup Condition="'$(RuntimeIdentifier)' == 'win-arm64'">
    <DefineConstants>$(DefineConstants);WINDOWS_ARM64</DefineConstants>
  </PropertyGroup>
  <PropertyGroup Condition="'$(RuntimeIdentifier)' == 'win-x64'">
    <DefineConstants>$(DefineConstants);WINDOWS_AMD64</DefineConstants>
  </PropertyGroup>
  <PropertyGroup Condition="'$(RuntimeIdentifier)' == 'osx-arm64'">
    <DefineConstants>$(DefineConstants);OSX_ARM64</DefineConstants>
  </PropertyGroup>
  <PropertyGroup Condition="'$(RuntimeIdentifier)' == 'osx-x64'">
    <DefineConstants>$(DefineConstants);OSX_AMD64</DefineConstants>
  </PropertyGroup>
```

上のようにcsprojでDefineを使っておくと、ビルド時にDefineの値によって
有効なソースコードが変わるので、環境ごとにビルドされるものが変わります。

```cs
#if WINDOWS_ARM64 || WINDOWS_AMD64
    // [LibraryImport("C:/Windows/SysWOW64/msvcrt.dll")]
    [LibraryImport("msvcrt.dll")]
// libc.soはシンボリックリンクのため、直接指定する。
#elif LINUX_AMD64
// debianは/lib/x86_64-linux-gnu/libc.so.6 osによってパスが違う。
    [LibraryImport("/lib/x86_64-linux-gnu/libc.so.6")]
#elif LINUX_ARM64
    [LibraryImport("/lib/aarch64-linux-gnu/libc.so.6")]
#endif

```

これはMicrosoftのPowershellにも使われているソースコードで、
十二分にプロダクションにも使っていける方法です。

こちらの方が妥当と思うならこちらを使ってください。
こちらの方が各アーキテクトごとに不要な処理をビルドしなくなるので、
バイナリ自体の大きさは抑えられると思います。

## powershellのソースコード

先ほど作った、rustの共有ライブラリと、csharpのdllをまとめてPowershell
から呼び出せるようにします。
ここではPowershellのスクリプトモジュールを作ります。

スクリプトモジュールのディレクトリ配下に
share_libというディレクトリを作り、
share_lib/architecure/共有ライブラリ
という風にrustで作った共有ライブラリを配置します。

同様に
csharpで作ったdllもchsarp_dllというディレクトリを作り
charp_dll/targetframeworkのバージョン/作ったdll
というふうにcsharpのdllを配置します。

そして下のスクリプトモジュールを書いてください。

```powershell:PowershellInvokeRust.psm1
$path = (Split-Path $MyInvocation.MyCommand.Path -Parent)

$path = Join-Path $path "csharp_dll"

$PSversion = $PSversiontable.PSVersion

$src = ""

# windows only
if ($PSVersion.Major -eq 5 -and $PSVersion.Minor -ge 1) {
    $path = Join-Path $path "net48"
} elseif ($PSVersion.Major -eq 7 -and $PSVersion.Minor -ge 3) {
    $path = Join-Path $path "net7.0"
} elseif ($PSVersion.Major -eq 7 -and $PSVersion.Minor -eq 2) {
    $path = Join-Path $path "net6.0"
} else {
    # not supported powershell
    # throw Error
}

$src = Join-Path $path "lib.dll"
# load csharp library.
Add-Type -Path $src

# specified architecture
[SystemArch] $global:os_type = [SystemArch]::Unknown
if ($PSVersion.Major -eq 5 -and $PSVersion.Minor -ge 1) {
    $global:os_type = [SystemArch]::X64
} else {
    if ($PSversiontable.OS -match "Darwin" -or $PSversiontable.OS -match "Linux") {
        $unamem = (uname -m)

        if ($unamem -match "x86_64") {
            $global:os_type = [SystemArch]::X64
        } elseif ($unamem -match "aarch64") {
            $global:os_type = [SystemArch]::Aarch64
        }
        # x86_64
    } elseif ($PSversiontable.OS -match "Windows") {
        $unamem = systeminfo.exe | Where-Object {$_ -match "System Type:"}

        if ($unamem -match "x64") {
            $global:os_type = [SystemArch]::X64
        } elseif ($unamem -match "ARM") {
            $global:os_type = [SystemArch]::Aarch64
        }
    }
}

function Measure-Add {

    param (
        [Parameter(Mandatory=$true)]
        [Int32] $x,
        [Parameter(Mandatory=$true)]
        [Int32] $y
    )

    if ($PSversiontable.PSVersion.Major -eq 5 -and $PSversiontable.PSVersion.Major -ge 1) {
        [lib.Math]::add_x86_64_pc_windows_msvc($x, $y)
    } elseif ( $PSversiontable.OS -match "Windows" -and $global:os_type -eq [SystemArch]::X64) {
        [lib.Math]::add_x86_64_pc_windows_msvc($x, $y)
    } elseif ( $PSversiontable.OS -match "Windows" -and $global:os_type -eq [SystemArch]::Aarch64) {
        [lib.Math]::add_aarch64_pc_windows_msvc($x, $y)
    } elseif ( $PSversiontable.OS -match "Darwin" -and $global:os_type -eq [SystemArch]::X64) {
        [lib.Math]::add_x86_64_pc_apple_darwin($x, $y)
    } elseif ( $PSversiontable.OS -match "Darwin" -and $global:os_type -eq [SystemArch]::Aarch64) {
        [lib.Math]::add_aarch64_pc_apple_darwin($x, $y)
    } elseif ( $PSversiontable.OS -match "Linux" -and $global:os_type -eq [SystemArch]::X64) {
        [lib.Math]::add_x86_64_unknown_linux_gnu($x, $y)
    } elseif ( $PSversiontable.OS -match "Linux" -and $global:os_type -eq [SystemArch]::Aarch64) {
        [lib.Math]::add_aarch64_unknown_linux_gnu($x, $y)
    }
}
```

こうするとこのスクリプトモジュールが

```powershell
Import-Module スクリプトモジュール名
```

とした時にcsharpのdllを読み込むことにより、
csharpのdllに書かれているオブジェクトが読み込まれます。

これらのオブジェクトを今回実装したいMeasure-Addという関数が呼び出された時に
呼び出される作りになっているため、
powershellからcsharp, rustというふうに処理が呼び出せるということです。

## github action

さてこれらの環境をgithubでリリースタグがpushされた時にビルド、
PSgalleryにデプロイされるようにCIを組んでみましょう。

CIの経験無いとかなり難しい構成になっていますが、これ以上簡単にできません...
Powerhshellチームが頑張らないとダメですねこれは。

まず、PSGalleryに登録するために
NUGET_APIのAPIキーを取得して下さい。
PSGalleryとnuget.orgは同じAPIキーを使います。

これをgithub actionで実行できるようにsecretに登録しておいてください。

```bash
gh secret set NUGET_API_KEY -b "apiキーの値"
```

下のようなgithubactionを書きます。

タグにvという接頭辞がついているものがpushされた時に、
このactionは動きます。

```yml:.github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  # for create release tag
  contents: write

jobs:
  rust-build:

    env:
      CARGO_TERM_COLOR: always
    strategy:
      matrix:
        os: [ubuntu-20.04, windows-2019, macos-12]
    # 必要になる
    runs-on: ${{ matrix.os }}

    steps:
    - name: specific running os arch.
      id: osarch
      shell: bash
      run: |
        target=""
        suffix=""
        ext=""
        if [ "${{ matrix.os }}" = "ubuntu-20.04" ]; then
          target="x86_64-unknown-linux-gnu"
          suffix="lib"
          ext="so"
        elif [ "${{ matrix.os }}" = "windows-2019" ]; then
          target="x86_64-pc-windows-msvc"
          ext="dll"
        elif [ "${{ matrix.os }}" = "macos-12" ]; then
          target="x86_64-apple-darwin"
          suffix="lib"
          ext="dylib"
        else
          echo "error"
        fi
        echo "target=${target}" >> "$GITHUB_OUTPUT"
        echo "suffix=${suffix}" >> "$GITHUB_OUTPUT"
        echo "ext=${ext}" >> "$GITHUB_OUTPUT"

    - uses: actions/checkout@v4
    - name: Build
      run: cd rust_src && cargo build --target ${{ steps.osarch.outputs.target }} --verbose --release

    - name: upload share_library
      uses: actions/upload-artifact@v3
      with:
        name: ${{ steps.osarch.outputs.target }}
        path: rust_src/target/${{ steps.osarch.outputs.target }}/release/${{ steps.osarch.outputs.suffix }}rust_src.${{ steps.osarch.outputs.ext }}
        if-no-files-found: error

    # - name: Run tests
    #   run: cargo test --verbose

  csharp-build:
    # 必要になる
    runs-on: windows-2019

        # id: tag_version
        # run:  |
        #   $tag = $env:GITHUB_REF -replace 'refs/tags/', ''
        #   echo $tag
        #   echo "tag=${tag}" >> $env:GITHUB_OUTPUT
    steps:
    # Install the .NET Core workload
    - name: Install .NET Core
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 7.0.x

    - uses: actions/checkout@v4
    - name: Build
      run: cd csharp_src && dotnet build --configuration Release

    - name: upload share_library net6.0-artifact
      uses: actions/upload-artifact@v3
      with:
        name: net6.0-artifact
        path: csharp_src/lib/bin/Release/net6.0/lib.dll
        if-no-files-found: error
        
    - name: upload share_library net7.0-artifact
      uses: actions/upload-artifact@v3
      with:
        name: net7.0-artifact
        path: csharp_src/lib/bin/Release/net7.0/lib.dll
        if-no-files-found: error

    - name: upload share_library net48-artifact
      uses: actions/upload-artifact@v3
      with:
        name: net48-artifact
        path: csharp_src/lib/bin/Release/net48/lib.dll
        if-no-files-found: error

  powershell_release:
    runs-on: windows-2019
    needs: [rust-build, csharp-build]


    steps:
      - name: Setup repo
        uses: actions/checkout@v4

      - name: Get Tag Version
        id: tag_version
        run:  |
          $tag = $env:GITHUB_REF -replace 'refs/tags/', ''
          echo $tag
          echo "tag=${tag}" >> $env:GITHUB_OUTPUT
      #  tempalteから.psd1ファイルを作成する。
      - name: Generate PSD1
        run: |
          $tagVersion = "${{ steps.tag_version.outputs.tag }}"
          echo $tagVersion
          $tagVersion = $tagVersion -replace '^v', ''  # 先頭の 'v' を削除
          $content = Get-Content PowershellInvokeRust/PowershellInvokeRust.psd1 -Raw
          # replace 
          $content  -replace '#tagVersion', "$tagVersion" |
            Set-Content -Path PowershellInvokeRust/PowershellInvokeRust.psd1
        
      - name: Download artifacts x86_64-unknown-linux-gnu
        uses: actions/download-artifact@v3
        with:
          name: x86_64-unknown-linux-gnu
          path: ./PowershellInvokeRust/share_lib/x86_64-unknown-linux-gnu/

      - name: Download artifacts x86_64-pc-windows-msvc
        uses: actions/download-artifact@v3
        with:
          name: x86_64-pc-windows-msvc
          path: ./PowershellInvokeRust/share_lib/x86_64-pc-windows-msvc/

      - name: Download artifacts x86_64-apple-darwin
        uses: actions/download-artifact@v3
        with:
          name: x86_64-apple-darwin
          path: ./PowershellInvokeRust/share_lib/x86_64-apple-darwin/

      - name: Download artifacts net6.0
        uses: actions/download-artifact@v3
        with:
          name: net6.0-artifact
          path: ./PowershellInvokeRust/csharp_dll/net6.0/

      - name: Download artifacts net7.0
        uses: actions/download-artifact@v3
        with:
          name: net7.0-artifact
          path: ./PowershellInvokeRust/csharp_dll/net7.0/

      - name: Download artifacts net48
        uses: actions/download-artifact@v3
        with:
          name: net48-artifact
          path: ./PowershellInvokeRust/csharp_dll/net48/

      - name: Publish Module
        env:
          NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
        run: Publish-Module -Path ./PowershellInvokeRust -NuGetApiKey  "$env:NUGET_API_KEY"

      - name: Release tag
        uses: softprops/action-gh-release@v1
        if: startsWith(github.ref, 'refs/tags/')
        with:
          body: |
            Changes in this Release
            - First Change
            - Second Change
          draft: false
          prerelease: false

```

コードの説明をしていきます。
jobsのrust-buildを見てください。
ここではubuntu, mac, windowsの各環境ごとにビルドを
行いその環境ごとの共有ライブラリを作成しています。
作成した後は、のちのjobで使うために一度、upload-artifactでgithubにアップロードしています。

csharpも同様にビルドしたのちにdotnet frameworkのバージョンごとにupload-arifactでアップロードします。

最後にPowershellのジョブを見てみましょう。
ここではrustとcsharpのビルドジョブが終わってから、その成果物をまとめて、
PSGalleryにデプロイしています。

また、PSGalleryにモジュールを登録するためにpsd1ファイルを作成する必要があり、
その作成をこのjob内で行ます。

psd1は作成するgithub actionがあったらよかったのですが、Powershellのチームが作っていないのと、
github action内で動的に作成するのはかなり難しいため、ここではpsd1の雛形を作成して、
github actionから必要な項目を修正することにより、PSGalleryに登録すべき内容に修正します。

psd1は下のように書きます。

```powershell:PowershellInvokeRust/PowershellInvokeRust.psd1

# 

@{
    ModuleVersion = '#tagVersion'
    Author = 'Otogawa Katsutoshi'
    Copyright = 'Otogawa Katsutoshi. All rights reserved.'
    # Supported PSEditions
    CompatiblePSEditions = 'Core', 'Desktop'
    PowerShellVersion = '5.1'
    Description = 'Powershell Invoke rust sample'
    GUID = 'f3192037-2de7-4995-bb20-83e4ea6fc3ff'
    ModuleToProcess = 'PowershellInvokeRust.psm1'
    FunctionsToExport = 'Measure-Add'

    PrivateData = @{
        PSData = @{
            Tags = 'rust'
            ProjectUri = 'https://github.com/KatsutoshiOtogawa/PowershellInvokeRust'
            LicenseUri = 'https://github.com/KatsutoshiOtogawa/PowershellInvokeRust/blob/v#tagVersion/PowershellInvokeRust/LICENSE'
            ReleaseNotes = 'Release notes for version 1.0'
        }
    }
}

```

psd1には頻繁に変更する内容と変更しない内容がありますが、リリースするたびに変更するべき
内容があります。
ModuleVersion, LicenseUri, ReleaseNotesです。

ModuleVersionは当たり前ですが、リリースするたびにバージョンが更新されますし、
LicenseUriはリリースされるたびにそのリリースタグを参照するLicenseに誘導する必要があります。
そうしないと途中でLicense条件が変わったのかわからないのでトラブルになります。

ReleaseNotesは面倒くさいので今回はやりませんが、
softprops/action-gh-releaseのReleaseNotesと同じ内容のものをどこかでファイルとして
持っておいて、それを読み込みで編集してください。

今回は#tagVersionをModuleのバージョンの置換変数として使います。
この文字列をgithub actionから動的に変更してくだださい。

処理の例はGenerate PSD1とGet TagVersionという名前のstepを見て参考にしてください。

その次に各ジョブでアップロードしたupload-arifactをdownload-artifactで指定の場所にダウンロードすることにより、Powershellのモジュール実行時にパスが解決されるようにします。

最後にPowershellのモジュールの形式としてまとまったので、
Github actionからPublish-Moduleを実行することにより、PSGalleryに登録します。

## 本格的に描きたい場合

win32api周りを参考にrustのffiの部分の引数や戻り値を考えると良い。

CreateFileAやら、 MessageBox, backup周りのapiが参考になると思う。

## まとめ

理屈がわかれば呼び出せることはわかったと思います。

poweshellから共有ライブラリ直接呼び出せるようにならんかな？と思うのと、Powershellのgithub actionいい加減に対応しろよというのと、windowsの共有ライブラリとcsharpのライブラリを同じ拡張子のdllにしたのはどう考えてもMSのミスとかまぁいろいろ思うことはありますなぁ...

ソースコードのライセンスはGPL3です。
好きに使ってください。
