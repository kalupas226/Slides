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

- TCA では v0.23.0 から `ViewStore.send(_:while:)` というものが導入されている(現在時点では Beta ）
- これを refreshable 内で利用すると TCA で refreshable が上手く扱える
- 「Async Refreshable: Composable Architecture」という Point-Free 内のエピソードをもとに、<br />TCA でどのように refreshable が扱えるか見てこうと思います

---

# Refreshable with TCA を理解するために利用する例

<div class="flex justify-center mt-5">
  <img src="/images/refreshable.gif" width=200>
</div>

---

# API Client 的部分

```swift
struct FactClient {
  var fetch: (Int) -> Effect<String, Error>
  struct Error: Swift.Error, Equatable {}
}

extension FactClient {
  static let live = Self(
    fetch: { number in
      URLSession.shared.dataTaskPublisher(
        for: URL(string: "http://numbersapi.com/\(number)/trivia")!
      )
      .map { data, _ in String(decoding: data, as: UTF8.self) }
      .catch { _ in
        Just("\(number) is a good number Brent")
          .delay(for: 1, scheduler: DispatchQueue.main)
      }
      .setFailureType(to: Error.self)
      .eraseToEffect()
    }
  )
}
```

---

# State, Action, Environment

```swift
struct PullToRefreshState: Equtable {
  var count = 0
  var fact: String?
}

enum PullToRefreshAction: Equatable {
  case cancelButtonTapped
  case decrementButtonTapped
  case incrementButtonTapped
  case refresh
  case factResponse(Result<String, FactClient.Error>)
}

struct PullToRefreshEnvironment {
  var fact: FactClient
  var mainQueue: AnySchedulerOf<DispatchQueue>
}
```

---

# Reducer

```swift
refreshReducer = Reducer<PullToRefreshState, PullToRefreshAction, PullToRefreshEnvironment> 
{ state, action, environment in
	struct CancelId: Hashable {}
	switch action {
	case .decrementButtonTapped:
		state.count -= 1
		return .none
	case .incrementButtonTapped:
		state.count += 1
		return .none
	case let .factResponse(.success(fact)):
		state.fact = fact
		return .none
	case .factResponse(.failure):
		return .none // TODO: エラーハンドリング
	case .refresh:
		return environment.fact.fetch(state.count)
			.receive(on: environment.mainQueue)
			.catchToEffect(PullToRefreshAction.factResponse)
			.cancellable(id: CancelId())
	case .cancelButtonTapped:
		return .cancel(id: CancelId())
	}
}
```

---

# View(store 宣言部分)

```swift
struct PullToRefreshView: View {
	let store: Store<PullToRefreshState, PullToRefreshAction>

	var body: some View {
		// ...
	}
}
```

---

# View(body)

```swift
var body: some View {
	WithViewStore(self.store) { viewStore in
		 List {
			HStack {
				Button("-") {
					viewStore.send(.decrementButtonTapped)
				}
				Text("\(viewStore.count)")
				Button("+") {
					viewStore.send(.incrementButtonTapped)
				}
			}
			.buttonStyle(.plain)

			 if let fact = viewStore.fact {
				 Text(fact)
			 }
		}
		.refreshable {
			viewStore.send(.refresh)
		}
	}
}
```

---

# Preview

```swift
struct PullToRefreshView_Previews: PreviewProvider {
	static var previews: some View {
		PullToRefreshView(
			store: .init(
				initialState: .init(),
				reducer: pullToRefreshReducer,
				environment: .init(
					fact: .live,
					mainQueue: .main
				)
			)
		)
	}
}
```

---

# 動かしてみる

<div class="flex justify-center mt-5">
  <img src="/images/refreshable_not_async.gif" width=200>
</div>

---
layout: center
class: text-center
---

# なんか良さそう？？

---
