---
title: "let が const だった — フロントエンドエンジニアが Swift で踏んだ7つの落とし穴"
emoji: "🪤"
type: "tech"
topics: ["swift", "swiftui", "ios", "javascript", "typescript"]
published: false
---

## はじめに

Web フロントエンド（メインは Vue / TypeScript）を中心に8年ほど書いてきましたが、iOS アプリ開発は未経験でした。広告なしで子どもの写真をすぐ撮りたい、撮った写真はそのまま家族用の保存先に入れたい、という個人的な要求があって、カメラアプリを作り始めています。

React Native も検討しましたが、カメラとリアルタイムのフィルタ処理が中心になるアプリなので、資料の厚みを優先して Swift / SwiftUI に決めました。

この記事は、その最初の数ステップで **実際に間違えたこと** の記録です。「こう書けばいい」ではなく「こう書いて失敗した」という形で並べています。同じく JS/TS から来る人なら、たぶん全部踏む地雷です。

---

## 1. `let` と `var` が JS と逆だった

最初に踏んだのがこれです。

```swift
let currentIndex = 0
currentIndex = 1   // ❌ Cannot assign to value: 'currentIndex' is a 'let' constant
```

JS の感覚で「とりあえず `let`」と書いたら、再代入でコンパイルエラーになります。対応表はこうです。

| Swift | JS |
|---|---|
| `let` | **`const`** |
| `var` | `let` |

Swift に `const` というキーワードはありません。`let` がその役割です。

紛らわしいのは、**キーワードの名前だけが入れ替わっていて、推奨される使い方は同じ** という点です。Swift の標準的な書き方も「まず `let`、本当に変わるものだけ `var`」なので、TS で「まず `const`」としてきた習慣はそのまま通用します。手が覚えている文字だけが違います。

---

## 2. `import` の範囲を勘違いした

`cannot find 'AVCaptureSession' in scope` で止まりました。同じプロジェクト内に書いたのに、なぜ見つからないのか。

整理するとこうです。

- **同じモジュール（自分のアプリ）内の別ファイル** → `import` 不要。そのまま使える
- **フレームワーク**（`AVFoundation`, `SwiftUI`, `UIKit` など） → **`import` が必須**

最初は「ファイルを分けたから import がいるのでは」と考えたのですが、逆でした。自作の `struct` や `extension` はファイルをまたいでも何もせず使えて、Apple が提供する型だけが明示的な宣言を要求します。

ES Modules では自作のものも常に `import` が要るので、ここは感覚が反転します。

ちなみに Xcode 16 以降のプロジェクトは **synchronized folder** という仕組みになっていて、フォルダにファイルを置くだけでビルド対象に入ります。以前は「Add Files to...」でプロジェクトに登録し、Target Membership にチェックを入れる必要があったそうで、そこは楽になったようです。

`cannot find X in scope` は、体感で7割が import 漏れでした。

---

## 3. 呼び出し側にラベルの選択権がない

似たような呼び出しなのに、ラベルが付くものと付かないものがあります。

```swift
Text("Hello")                      // ラベルなし
Color(hex: "FA9C4A")               // ラベルあり
.frame(width: 100, height: 100)    // ラベルあり
```

最初は「省略できるときは省略してよい」のだと考えていたのですが、違いました。**付けるか付けないかは、関数を宣言した側が決めています。** 呼び出す側に選択権はありません。

```swift
func noLabel(_ x: Int) { }
noLabel(x: 5)      // ❌ extraneous argument label 'x:' in call

func withLabel(x: Int) { }
withLabel(5)       // ❌ missing argument label 'x:' in call
```

両方向ともエラーになります。宣言で `_` を書くと「ラベルなし」が確定し、書かなければ「ラベルあり」が確定します。

JS には位置引数しかないので、名前付き引数が欲しければオブジェクトで代用します。

```js
frame({ width: 100, height: 100 })
```

Swift がこれを言語機能として持ち、しかも設計者側に固定させているのは、**呼び出し箇所が文章として読めること** を重視しているからです。

```swift
Text("Hello")                          // Text(content:) だと冗長
AVCaptureDevice.default(for: .video)   // for があって初めて意味が通る
session.canAddInput(input)             // input は自明なのでラベル不要
```

自分で API を書くときにも同じ判断を迫られます。`Color(hex:)` を定義したときは「`Color("FA9C4A")` では16進数だと分からない」と考えてラベルを付けました。

---

## 4. 値が「ある」ことを型で保証させられた

カメラを探すコードで、いきなり手が止まりました。

```swift
let device = AVCaptureDevice.default(for: .video)
let input = try AVCaptureDeviceInput(device: device)   // ❌
```

`AVCaptureDevice.default(for:)` が返すのは `AVCaptureDevice` ではなく **`AVCaptureDevice?`** です。末尾の `?` が付いた型は、`?` の付かない型と **別物** として扱われます。そのままでは引数に渡せません。

JS では「`null` になりうる値」も、そうでない値も、同じ変数に入ります。問題が表面化するのは実行時です。

```js
const device = findDevice()
device.configure()   // device が null なら、ここで初めて落ちる
```

Swift はこれをコンパイル時に止めます。値を取り出すには、**必ず「無かった場合」を書かされます**。

```swift
guard let device = AVCaptureDevice.default(for: .video) else {
    throw CameraError.noDevice
}
// ここから先の device は AVCaptureDevice（? が取れている）
```

TypeScript の `strictNullChecks` に似ていますが、決定的に違う点があります。**TS はオプトインで、`any` を挟めば逃げられます。Swift の Optional は言語の前提** なので、迂回路がありません。

### `guard let` と `if let` — 取り出した値の寿命が違う

剥がす構文は2つあり、**値の生きる範囲が違います**。

```swift
if let device = AVCaptureDevice.default(for: .video) {
    // ここでしか使えない
}
// ここでは使えない

guard let device = AVCaptureDevice.default(for: .video) else {
    throw CameraError.noDevice
}
// ここから関数の最後まで使える
```

`if let` は波括弧の中だけ。`guard let` は **抜けた後もそのまま使い続けられます**。「ここを通過したなら値がある」と言い切れるので、その先で生きているのが理屈に合っています。

そしてもう1つ、`guard` には制約があります。**`else` の中では必ずスコープを抜けなければなりません。**

```swift
guard n > 0 else {
    print("ダメでした")      // ❌ 'guard' body must not fall through
}
```

`return` / `throw` / `continue` / `break` のいずれかが要ります。**「通過したなら条件は成立している」ことをコンパイラが保証するための縛り** です。

JS/TS でも早期 return は書けますし、TS なら制御フロー解析で型も絞られます。

```ts
if (!device) return
device.configure()   // ここでは non-null に絞られている
```

効果は近いのですが、**`return` を書き忘れても TS は止めてくれません**。条件が絞られないまま素通りするだけです。Swift は構文そのものが抜けることを要求するので、書き忘れが起こりません。

この違いのおかげで、失敗条件を関数の先頭へ並べて、本処理をインデントなしで下へ流す書き方が自然になります。

```swift
guard let device = ... else { throw CameraError.noDevice }
let input = try AVCaptureDeviceInput(device: device)
guard session.canAddInput(input) else { throw CameraError.cannotAddInput }
session.addInput(input)
```

関数の頭だけ読めば「何が失敗しうるか」が分かる形です。`if` のネストで書くと、本当にやりたい処理が一番深い場所へ沈みます。

そしてこれは単なる厳しさではなく、実際に必要でした。iOS シミュレータにはカメラが存在しないので、この関数は本当に `nil` を返します。JS の感覚のまま書いていたら、シミュレータで実行した瞬間に落ちていたはずです。

---

## 5. `!` は「保証するから、外れたら落ちていい」

Optional を触っていると `?` と `!` が頻繁に出てきますが、この2つは **Optional 以外の場所にも同じ意味で現れます**。

| | 安全（失敗を許容） | 強制（失敗したらクラッシュ） |
|---|---|---|
| Optional | `value?` | `value!` |
| 型変換 | `as?` | `as!` |
| エラー | `try?` | `try!` |

3つとも「`?` は失敗を `nil` として受け取る」「`!` は失敗したらその場で落ちる」という同じ規則です。

```swift
let layer = view.layer as! AVCaptureVideoPreviewLayer   // 型が違えばクラッシュ
let data = try? Data(contentsOf: url)                    // 失敗したら nil
```

`!` を使ってよいのは **「失敗したとしたら、それは自分のコードの誤り」** という場面に限られます。プレビュー用のレイヤーを取り出す箇所では `as!` を使いました。レイヤーの型を自分で指定しているので、違う型が来ることはあり得ないからです。

逆に、カメラやネットワークのように **外部要因で失敗しうる処理** に `!` を使うと、ユーザーの環境でアプリが落ちます。

JS の `?.` は `?` 側に近いものですが、`!` に相当するものはありません。TypeScript の `!`（non-null assertion）は型検査を黙らせるだけで、**実行時には何も起きません**。Swift の `!` は実行時に本当にチェックして、外れたら停止します。同じ記号でも意味が違います。

---

## 6. `struct` はコピーされる

カメラを管理するクラスを最初 `struct` で書こうとして、`class` に変えました。

JS ではオブジェクトはすべて参照です。

```js
const a = { count: 0 }
const b = a
b.count = 1
console.log(a.count)   // 1 — 同じものを指している
```

Swift では `struct` が **コピー** されます。

```swift
struct Counter { var count = 0 }

var a = Counter()
var b = a
b.count = 1
print(a.count)   // 0 — 別物になっている
```

カメラセッションのような「ひとつしか存在してはいけないもの」を `struct` で持つと、**どれが本物のセッションなのか分からなくなります**。だから `class`（参照型）にする必要がありました。

ここまでは「そういう仕様なのだな」で済むのですが、SwiftUI に入ると話が変わります。**SwiftUI の `View` は全部 `struct` です。**

```swift
struct ContentView: View {   // class ではない
    var body: some View { ... }
}
```

Vue ではコンポーネントのインスタンスが生き続け、リアクティブな値が変わるとレンダー関数だけが再実行されます。SwiftUI は違います。**構造体そのものが何度も作り直されては捨てられる** という前提で動いています。軽い値型だからこそ、それが許されている設計です。

画面が作り直される前提なら、状態はどこに置けばよいのか。そこにも独自の仕組みがあるのですが、長くなるので別の記事にします。

---

## 7. `defer` は `finally` と似て非なるものだった

処理の途中で抜けると、開いたものが閉じられない。これ自体は JS でも同じです。

```js
lock.acquire()
doSomething()    // ここで例外が出ると
lock.release()   // 到達しない
```

カメラセッションの設定も同じ形をしています。`beginConfiguration()` と `commitConfiguration()` で挟んだ間の変更が、`commit` の時点で一括反映される仕組みです。エラーを呼び出し元へ伝えようと `throws` に変えたところ、途中で抜ける経路が増えて `commit` に届かなくなりました。

JS なら `try/finally` で囲むところです。Swift には `defer` があります。

```swift
session.beginConfiguration()
defer { session.commitConfiguration() }
```

ここまでは素直に対応しているのですが、**`defer` には `finally` にない規則が3つ** ありました。

### ブロックで囲まない

`try/finally` は本体全体を囲む必要があり、インデントが一段深くなります。閉じる処理も本体の下へ離れてしまうので、間に100行入ると対になっているか確認しづらくなります。

`defer` は **「開く処理」の真下に「閉じる処理」を書けます**。囲まないのでインデントも増えません。

### その行を通過しないと登録されない

ここが一番の違いでした。`finally` は構造で決まります。`try` ブロックへ入った以上、`finally` は必ず走ります。

`defer` は **実行された時点で初めて「あとでこれを実行する」と予約されます**。

```swift
func f(_ bail: Bool) {
    print("開始")
    if bail {
        return                   // defer 行に到達していない
    }
    defer { print("defer") }
    print("終了")
}
```

`bail` が `true` のとき、`defer` の中身は実行されません。予約自体がされていないからです。

この性質は置き場所に直結します。

```swift
// ✓ 正しい
guard let device = ... else { throw CameraError.noDevice }  // ここで抜けても
session.beginConfiguration()                                 // まだ開いていないし
defer { session.commitConfiguration() }                      // 予約もされていない

// ✗ 前に出すと
defer { session.commitConfiguration() }                      // 先に予約してしまう
guard let device = ... else { throw CameraError.noDevice }   // ここで抜けたとき
session.beginConfiguration()                                 // 開いてもいない設定を閉じにいく
```

`try/finally` なら構造上そもそも起こらない間違いが、`defer` では起こりえます。**柔軟なぶん、対応が取れているかの責任が書き手へ移っています。**

### 関数ではなくスコープ単位

`defer` はループの本体にも書けて、**1周ごとに** 実行されます。

```swift
for i in 1...2 {
    print("ループ \(i) 開始")
    defer { print("ループ \(i) defer") }
    print("ループ \(i) 本体")
}
```

```
ループ 1 開始
ループ 1 本体
ループ 1 defer
ループ 2 開始
ループ 2 本体
ループ 2 defer
```

同じスコープに複数書くと、**後に登録したものから** 実行されます。A → B → C の順に開いたものを C → B → A で閉じる、という流れがそのまま書けます。

---

## まとめ — コンパイラが止めてくれる境界

並べてみて気づいたのは、7つがきれいに2種類へ分かれることでした。

**型として間違いを表現できるもの**（1〜4）は、その場でコンパイルエラーになります。メッセージを読んで直すだけなので、むしろ楽でした。Swift の型検査は TS よりかなり厳しく、握りつぶす余地が少ないぶん、間違いがそこで止まります。

**型は合っているのに意図と違うもの**（5〜7）に時間を取られました。

- `as!` / `try!` — 型は通る。落ちるのは実行時
- `struct` のコピー — コピーが作られても型としては正しい
- `defer` の置き場所 — 1行前後させても型は通る

どれも「書けてしまう」ので、動かしてみるまで分かりません。型システムが守ってくれる範囲と、そこから先の境界がどこにあるかは、こういう失敗を通してしか掴めない気がしています。

次はシャッターで実際に撮影する部分に入ります。また何か踏んだら書きます。
