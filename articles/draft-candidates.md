---
title: "【下書き】次回記事のストック"
emoji: "📝"
type: "tech"
topics: ["swift"]
published: false
---

:::message
1本目（`js-engineer-swift-pitfalls`）は「言語そのものの違い」7つに絞って公開する方針にしたため、
SwiftUI（フレームワーク）側の項目をここに退避しています。
今後のステップで同種の項目が溜まったら、2本目としてまとめます。

想定テーマ: **SwiftUI の前提がフロントエンドの常識と違ったところ**
:::

---

## `@State` を `struct` の外に書いて、画面が更新されなかった

一番気づきにくかったものです。

```swift
@State private var currentIndex = 0   // ← struct の外

struct ContentView: View {
    var body: some View { ... }
}
```

**コンパイルは通ります。ボタンを押しても何も起きません。**

SwiftUI は「この View がどの状態に依存しているか」を、**その View struct のプロパティを見て** 判断します。外側に書かれた変数は、SwiftUI から見るとどの View のものでもないので、値が変わっても再描画の対象になりません。

正しくは struct の中です。

```swift
struct ContentView: View {
    @State private var currentIndex = 0
    var body: some View { ... }
}
```

Vue で `ref` をコンポーネントの外（モジュールスコープ）に置いたようなものです。ただし Vue なら、リアクティブには動きます。全インスタンスで値が共有されるという別の問題は出ますが、少なくとも画面は更新されます。**SwiftUI はそもそも再描画の対象になりません。** エラーも警告も出ないまま、ただ何も起きませんでした。

> 1本目の「6. `struct` はコピーされる」の続きとして書くと、因果がつながる。

---

## `ZStack` の宣言順がそのままレイヤー順だった

カメラプレビューの上にフィルタの色を重ねる、つもりで書いたコードです。

```swift
ZStack {
    currentFilter.color      // ①
    CameraPreview(...)       // ②
    VStack { /* ボタン類 */ } // ③
}
```

カメラは映るのに、**矢印を押してもフィルタの色がまったく変わりません。**

`ZStack` は **先に書いたものが奥、後に書いたものが手前** です。つまり色を敷いた上からカメラ映像で塗りつぶしていました。名前のテキストだけが切り替わるので、しばらく「色の指定が間違っているのか」と別のところを疑っていました。

```swift
ZStack {
    CameraPreview(...)       // 奥
    currentFilter.color      // ← 映像の上に重なって着色される
    VStack { /* ボタン類 */ } // 手前
}
```

CSS の `z-index` のように数値で制御するのではなく、**宣言順がそのまま奥行き** です。
