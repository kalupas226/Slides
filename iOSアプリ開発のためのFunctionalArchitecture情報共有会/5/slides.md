---
theme: default
layout: intro
class: text-black-500 text-center
highlighter: shiki
---

# Refreshable API を<br />TCA で使う
## iOSアプリ開発のためのFunctional Architecture情報共有会5

<div class="abs-br m-6">
  <img src="/images/kalupas226.png" width=80>
</div>

---

# Refreshable API とは

- iOS 15 から利用できるようになった View Modifier
- SwiftUI なら簡単に Pull to Refresh を実現できる
- `refreshable` を利用するだけ
  - これが取る closure は async な処理を要求する

<br/>

```swift
List(mailbox.conversations) {
    ConversationCell($0)
}
.refreshable {
    await mailbox.fetch()
}
```
---

# TCA ではどのように refreshable を利用できるか

- TCA では v0.23.0 から `ViewStore.send(_:while:)` というものが導入されている
- これを refreshable 内で利用すると TCA で refreshable が上手く扱える
- 「Async Refreshable: Composable Architecture」という Point-Free 内のエピソードをもとに、<br />TCA でどのように refreshable が扱えるか見ていく

---

# Refreshable with TCA を理解するために利用する例

<div class="flex justify-center mt-5">
  <img src="/images/refreshable.gif" width=200>
</div>

---
