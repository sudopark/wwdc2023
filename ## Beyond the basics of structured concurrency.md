## eyond the basics of structured concurrency

: https://developer.apple.com/videos/play/wwdc2023/10170/


### Task hierarchy
- structured concurrency: concurrency 코드 제어를 돕는다
```swift

// structured concurrency
async let future ...

taskGroup.addTask {
	....
}

// unstructured concurrency
Task { ... }

Task.detached { ... }
```
- concurrency excuetion의 위의 경우에 만들어진다
- structured concurrency는 로컬 변수처럼 스코프내에서 존재하며 벗어날경우 취소된다.
-> 왠만하면 이거 써라
```swift
func makeSoup(order: Order) async throws -> Soup {
	// unstructure concurrency 만들고 대기
	let boilingPot = Task{ try await stove.boil() }
	let choppedIndigredints = Task { try await ... }
	let meat = Task { await ... }
	let soup = await Soup(meat: meat, ...)
	return await stove.cook(pot: boliingPot, soup: soup, duration: .minutes(10))
}

// 위를 다음과 같이 개선
func makeSoup(ordre: Order) async throws -> Soup {
	async let boilingPot = stove.boil()
	async let choppedIndigredints = ...
	async let meat = ...
	let soup = try await Soup(meat: meat, ...)
	return try await stove.cook(pot: boliingPot, soup: soup, duration: .minutes(10))
}
// 이 경우에는 3개의 task가 생성되었음을 알수있고 -> 병렬로 처리됨
// 또한 parent task와 관계가 있음
```

### Task cancellation
- stturctured concurrency는 스코프 벗어나면 자동으로 취소
- unstructured한 경우에는 수동으로 최소해야하고
- task group에서 parent가 캔슬되면 연쇄적으로 child들도 다 취소됨
```swift
func makeSoup(ordre: Order) async throws -> Soup {
	async let pot = stove.boil()

	guard !Task.isCancelled else {
		throw error...
	}

	// 혹은
	try Task.checkCancellation() // throw  CancellationError

	....
}

```
- checking for cancellation
```swift
// polling case
try Task.checkCancellation()
// or
Task.isCancelled

// Event-driven
withTaskCancellationHandler(operation:, onCancel:)
// cancel된 경우에 처리 가능
```
- cancellation 예제
```swift

// AsyncSequence의 iterator의 next function
public func next() async -> Order? {

	return await withTaskCancellationHandler {
		let result = await kitchn.generateOrder()
		// state에 의해 isRunning 여부를 판단 -> 시퀀스가 종료되었으면 nil반환해서 종료
		guard state.isRunning else {
			return nil
		}
		return result

	} onCancel: {
		// task 캔슬되면 이부분이 즉시 불리고 state는 취약한 상태임
		state.cancel()
	}
}


actor Cook {

	func handleShift<Orders>(
		orders: Orders
		) async throws 
		where Orders: AsyncSequence, Orders.Elemnt == Order 
		{

			for try await order in orders {
				// 요 loop에서는 취소되기 이전에 새 오더 이벤트를 받을수있음
				let soup = try await makeSoup(order)
				
			} // Task가 suspend 된 상태이상태인 경우 -> 명시적으로 취소 처리 못함

		}			
}

// 이를 위해 state를 actor로 바꾸면 좋지만 그러면 state의 모든 값에 접근하기가 어려워진다 -> 적절하지 않은 경우가 있음
// 또란 actor의 연산 순서도 장담할 수 없음 -> 그렇기에 cancel시 바로 state를 반영하기에는 부적절
// 그렇기때문에 아래 예제에서는 다음과같이 Swift Atomic package에 있는 Atomic을 사용하여 State를 정의, 혹은 디스패치큐나 락을 사용할수도있음
private final class OrderState: Sendable {

	private let protectedIsRunning = ManagedAtomic<Bool>(true)
	var isRunning: Bool {
		get { protectedIsRunning.load(ordering: .acquiring) }
		set { protectedIsRunning.store(newValue, ordering: .relaxed) }
	}

	func cancel() {
		isRunning = false
	}
}
```

### Task priority
- task는 parent task와 디폴트로 같음
- 또한 우선순위가 변경되면 이를 task tree의 차일드들에도 다 적용
- swift runtime이 우선순위 높은 작업이 먼저 실행되도록 수행

### Task group patterns
- 위 의 예제에서 chopping 동작은 병렬로 많이 수행될수 있고
- 너무 많이 child task들이 만들어지면 안좋다.
-> 이를 TaskGroup으로 개선
```swift
func chopIndgredient(_ ingredients: [Any Ingredient]) async -> [Any ChoppedIngredient] {

	return await withTaskGroup(
		of (ChoppedIngredient?).self), 
		returning: [any ChoppedIngredient].self 
		{ group in


			// 요 부분 수정해서 group에 들어가는 task 수 제한할수있음
			// 병렬로 재료 chop함
			for ingredient in ingredients {
				group.addTask { await  chop(ingredient) }
			}

			// 결과물 모음
			var sender: [any ChoppedIngredient] = []
			for await chopped in group {
				if let c = chopped {
					sender.append(c)
				}
			}

			return sender
		}

}
```
#### new API
- withDiscardingTaskGroup
- withThrowingDiscardingTaskGroup
- 자동으로 테스트 끝나면 취소해줌
- 또한 child task 하나라도 error throw한면 자동으로 취소함
- child task가 값을 리턴하지 않거나 사이드 이팩트만 만드는 usecase에 적당 -> 전체 결과를 모을 필요가 없어서, 이게 메모리 소모도 덜하다.

### Task local values
: https://developer.apple.com/documentation/swift/tasklocal
- task에 바인딩할수있고 task의 context 내에서 읽을 수 있는 값
- child task에도 전달됨?
- static 혹은 global 변수로 선언되어야함
- 래핑된 값이 optional 이라면 디폴트 값은 nil, 혹은 명시적으로 지정 가능
- task가 없는?경우는 디폴트값
- 명시적으로 값 세팅 못하고 특정 스코프에 바인딩 되어있어야함 -> 스코프가 끝나면 다시 원복?(이 스코프내에서 생성된 차일드 테스크들에게만 유지됨)
- detachedTask는 task-locals를 유지 안하지만 Task는 unstructured concurrency라 하더라도 별도 스레드에서 복사본을 지니고 있음
```swift
@TaskLocal
static var traceId: TraceId?

print("tradeId: \(tradeId)")	// traceId: nil

$traceId.withValue(1234) {		// bind the value
	print("traceId: \(traceId)")	// tradeId: 1234

	call()	// traceId: 1234

	Task {	// unstructured tasks do inherit task locals by copying
		call()	// tradeId: 1234
	}

	Task.detached {	// detached tasks do not inhreit task-local values
		call()	// tradeId: nil
	}
}
```
- task local storage에서 값을 식별하여 찾기 위해서 타입은 class 이여야 한다.

func call() {
	print("traceId: \(traceId)")	// 1234
}

#### swift log
- 생략

### Task traces
- task 동적 자세하게 보고싶으면 withSpan 참고