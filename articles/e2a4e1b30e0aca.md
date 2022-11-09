---
title: "windowsでのPHYチップの確認方法"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows", "powershell"]
published: true
---

NICに搭載されているPHYチップに対応した
ドライバーをとってくることにより、OSはそのNICを操作することができます。

Linuxだと、たいていのNICでちゃんと動きますが、
windowsやBSDだとちゃんと動くかわかりません。

そのためにあらかじめPHYチップを調べましょう。

windowsではpowershellで下記のように実行すると、
InterfaceDescriptionに書かれていることがあります。

```powershell
Get-Netadapter -Physical
```

これでもわからなければ下記のように実行してPNPデバイスに
それに近いチップが無いか確認してください。

自分の場合はServiceのプロパティにBCM43XXと書かれていました。

```powershell
$interfaceDescription = Get-Netadapter -Physical | Select-Object -ExpandProperty InterfaceDescription
Get-PnpDevice -Class Net | ?{$_.FriendlyName -eq $interfaceDescription } | Format-List *
```

これでもわからなかったり、しっかりした情報が欲しいなら、

Ubuntuのライブusbを接続して、そこからlscpiコマンドを実行して
PHYチップを調べることができます。

wslからだとlspciでもわかりません。
こういうところ本当にwindows...
