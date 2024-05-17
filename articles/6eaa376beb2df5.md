---
title: "Deno.argsとnodejsのprocess.argvの動きの違い"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno", "nodejs", "typescript", "javascript"]
published: true
---

Deno.argsとprocess.argvで配列がズレるので注意。

```ts:args.ts
import { argv as nodejs_argv } from 'node:process';

// それぞれコマンドラインから
// deno run args.ts aaa bbb
// と実行した結果です。--allow-readのようにパーミッションを与えても、deno tasksから実行しても配列に影響はありません。

console.log("Deno.args", Deno.args, "index:0", Deno.args[0], "index:1", Deno.args[1], "index:2", Deno.args[2], "index:3", Deno.args[3]);
// Deno.args [ "aaa", "bbb" ] index:0 aaa index:1 bbb index:2 undefined index:3 undefined

console.log("nodejs args", nodejs_argv, "index:0",nodejs_argv[0], "index:1",nodejs_argv[1], "index:2",nodejs_argv[2], "index:3",nodejs_argv[3]);
// nodejs args [ [Getter], [Getter], "aaa", "bbb" ] index:0 /home/vscode/.deno/bin/deno index:1 /workspaces/example/args.ts index:2 aaa index:3 bbb

```

nodejsの場合はランタイムと実行ファイルへのパスが配列に入ります。

同じようなものだと思って、単純に書き換えると配列のインデックスが２ずれます。

## まとめ

denoからnodejsのprocess.argvを呼べるので、既に動いているnodejsの処理ならnodejsからDenoに置き換えてもprocess.argvのままにした方が無難。

ただし、nodejsのprocess.argvには--allow-readのパーミッションが必要になります。

とりあえず動くので移行時に修正の優先順位としては高くは無い。
