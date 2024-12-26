---
title: "まとめ"
---

# さらにデバッガの機能を追加するには
ここまで、デバッガの基本的な機能を実装してきましたが、まだまだ足りない機能はいっぱいあります。たとえば、以下のようなものが考えられます。

- バックトレースを表示する
- 現在の変数の値を取得する
- VSCode などと連携して利用できるように [DAP](https://microsoft.github.io/debug-adapter-protocol/) に対応する

また、今回は defer やゴルーチンなどに対応していないので、そちらを実装していくのも面白いと思います。もっとデバッガを拡張したいと思った方は、ぜひこういった機能の実装にチャレンジしてみてください。

# 参考資料
- [CppCon 2018: Simon Brand “How C++ Debuggers Work”](https://www.youtube.com/watch?v=0DDrseUomfU)