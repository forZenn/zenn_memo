---
title: "powershellで動的にConfirmImpactを変えるバイナリModuleを作る。"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows", "dotnet", "powershell"]
published: true
published_at: 2023-01-24 19:00 
---

Powershellでは、ユーザーに確認要求を行う場合は
ConfirmImpactをHighにする[Confirm](https://learn.microsoft.com/ja-jp/powershell/scripting/developer/cmdlet/users-requesting-confirmation?view=powershell-7.3)とユーザーに確認要求ができる。

これはコマンドレット、関数の定義のところでAttributeとして設定する。
ConfirmImpactを動的にいじるには、コマンドレットのメタ情報を持っているCommandInfoを操作する必要がある。

CommandInfoは各コマンドレットのメンバー変数として定義されているが、
アクセス制限がInternalなため、普通のやり方ではアクセスできない。

このため、下記のケースの時にConfirmImpactを変えられないため、
下記の要件のコマンドを作るのが難しい。

1. 引数ごとに管理者権限が必要、不要と分けたい。
2. 管理者権限が不要な場合はそのまま実行、管理者権限が必要な場合は管理者権限＋画面確認Yesで実行。


CommandInfoからConfirmImpactを設定するには下記のように書こう。

```cs
// ConfirmImpactを動的にMediumに変えたい場合、
// ShouldProcessの前に下のように設定する。
var commandInfo = (CommandInfo)this.GetType().GetProperty("CommandInfo", BindingFlags.NonPublic | BindingFlags.Public | BindingFlags.Instance).GetValue(this);
((CommandMetadata)commandInfo.GetType().GetProperty("CommandMetadata", BindingFlags.NonPublic | BindingFlags.Public | BindingFlags.Instance).GetValue(commandInfo)).ConfirmImpact = ConfirmImpact.Medium;
```

## まとめ

要件的には割と自然な作りだしありうる仕様だが、
テクニカルな事をしないと動的に変えられないのは問題だなぁ...
プログラミングスキルというか、言語側が詳しいエンジニアかつ
powershellに詳しくないと無理やん。

しばらくして、なんも変わらんかったら
Powershellにissue立てるか...

また、余談ですが、Powershellのモジュールをスクリプトモジュールで作っていたとしても、
今回の仕様みたいに*動的にメタ情報をいじる事になった時点で、
私は速やかにバイナリモジュールに移る事を強く推奨します*。

*メタ情報をいじる程度の細かい制御が必要な時点でスクリプトモジュールは向いておりません*。
