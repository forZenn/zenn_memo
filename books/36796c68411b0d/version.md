---
title: "versionについて"
---


## Powershell2.0

Windows Server 2025で機能として削除が決まっており、
windows11 23H2現在ではデフォルトでは機能が有効にされているが、
危険なので無効にしておくことをお勧めします。

管理者権限でpowershellを開き下記のように実行すると、
Powershell2.0をアンインストールできます。

```powershell
Get-WindowsOptionalFeature -Online | ?{$_.FeatureName -match "MicrosoftWindowsPowerShellV2"} | Disable-WindowsOptionalFeature -Online
```

Powershellが一般的に使われるようになったのは5.0, 5.1以降のため、
ほとんどの企業ではアンインストールで困ることは無いだろう。

## Powershell 5.1

Powershellで現在最も多く使われているのがこのバージョンだ。
windows server 2025, windows11までのOSにデフォルトでインストールされている。

.netframework 4.7.2をベースに動いている。

dotnet core系には昔からある機能が使えず、
windowsでしか動作しない。
.netframeworkがベースのため、core系のPowershellと比べて測定するまでも無く、
かなり遅く、メモリの消費量も多い。

かと言って、core系と比べて動作が安定しているかというと怪しい。

新規で採用するメリットが全く無く、core系で動作しないときにスクリプトの一部で採用しよう。

もし、powershell 7.1ではない機能がある場合はPowershell 7.1からアプリケーションとして実行して
戻り値を取得したら良いだけである。

また、Powershell 5.1はソースコードが公開されていないため、
バグや不具合を見つけてもms待ちになるのでいつ対応してくれるのかわからないため、受け身になる可能性が高く、
できるだけ業務に依存させたくない。

このため現時点では、Corek系のPowershellの**最新のLTSを強く推奨する**。

windowsのシステムとしてはcore系と5.1系をちゃんと両立できるようになっているため、
衝突する不安もない。

## Powershell Core

Core系のPowershell。
dotnet coreが使われている。

現在はこちらがメインに開発が進められており、
Windowsだけではなく、macやlinuxでも動作する。

ランタイムとsdkが.netframeworkからdotnetに移行したことにより、
大幅に速度が改善している。


### install 方法

windows server 2025移行、windows11移行では下記のように
wingetを通常使ってインストールする。

```powershell
winget install Microsoft.PowerShell
```
