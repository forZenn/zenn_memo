---
title: "ループ内では複数のことをやらない"
---

そもそもawkとパイプでたいていの処理は吸収

速度やメモリのせいでどうしてもループ内で複数処理したい場合は、
そもそもbashなどのシェルスクリプトには向いてない処理です。
golangを使いましょう。

## awkとパイプでたいてい吸収できるはず
