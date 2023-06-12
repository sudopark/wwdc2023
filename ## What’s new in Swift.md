## What’s new in Swift

: https://developer.apple.com/videos/play/wwdc2023/10164/



### if/else switch statements to be used as expressions
- 어떤 변수 할당할때 분기가 필요하면 ternary expression 쎠야했었다 -> 분기 조건이 까다로우면 복잡해짐
```swift
lket bullet = 
	if isRoot && (count == 0 || !willExpand) { "" }
	else if count == 0 { "- "}
	else if maxDepth <= { "> "}
	else { "^" }
```
- 혹은 전역변수 초기화시에 바로 실행되는 클로저 안에 생성 구문을 넣어야 했다
```swift
// 기존
let some = {
	if .. { 0 }
	else { 1 }
}

// swift 5.9
let some = if .. { 0 } else { 1 }
```


### Result Builders
- faster type checking
- improved code completion
- more accurate error message
-> 유효하지 않은 코드 검증이 더 빨라졌음


### type parameter pack
- argument 길이에 따라 지원되는 제네릭(from swift 5.9)
- 복수의 개별 타입 파라미터가 packed된 새로운 개념 => "type parameter pack"
```swift
// argument 갯수에 따라 오버로드된 api
func foo<T>(_:) -> T
func foo<T1, T2>(_:_:) -> (T1, T2)
func foo<T1, T2, T3>(_:_:_:) -> (T1, T2, T3)
...

// 하나로 합쳐질수있음
func foo<each T>(_: repeat T) -> (repeat each T)

```
- 컴파일러는 전체 argument 수와 함께 개별 argument를 타입추론함


### Macro
- macro를 이용하여 언어 자체의 능력을 확장할 수 있음 + 불필요 코드 삭제
- swift에서 macro는 타입이나 함수와 비슷함 -> 임포트할 수 있고, 정의할수있음
- 다른 api들과 마찬가지로 package로 배포됨 -> swift open source 라이브러리에 몇개 있음
```swift
// power assert 라이브러리 내 assert macro 선언
@freestanding(expression)
public macro assert(_ condition: Bool) = #externalMacro(
	module: "PowerAssertPlugin",
	type: "PowerAssertMacro"
)
// 매크로가 값을 반환한다면 일반적인 arrow syntax를 반환?

// 사용
import PowerAssert
#assert(max(a, b) == c)
```
- 매크로 사용시 파라미터와 함께 타입체크됨
- 대부분의 매크로는 "external macro"로 정의된다. - external macro 타입은 다른 프로그램에 정의되고, 텀파일러 플러그인에서 동작한다.
- 컴파일러는 매크로 expression을 플러그인에 보내고, 플러그인은 소스코드를 반환, 이는 swift program과 통합된다.
- 매크로는 또한 하나의 추가정보를 받는데 이는 "role" 이다

-> 매크로 좀더 알고싶다면
Expand on Swift macros, Write Swift macros 참고


### object-c로 된 foundation 레거시 교체중
- 뭐 개선되었다고하는데 레거시가 원래 똥이였던거에 비하면 자랑할일인가..? 
- 생각해보면 맨날 wwdc에서 나오는 퍼포먼스 개선은 xcode 부터 시작해서 이런식임;;
- 몇년에 걸쳐 object-c 레거시 의존을 제거하고 swift 환경으로 변환하는 사실은 배울점이 있다 


### OwnerShip
- 옵트인으로 적용 가능
- 프로그램 내에서 어떤 코드가 값을 소유하고있냐를 컨트롤할수있음
```swift
// reference type같이 동작한다 -> class로 변경시 race condition의 타겟이 됨
// struct로 한다 하여도  mutatble한 상태라 버그가 발생할수있음
struct FileDescriptor { 
	private var fd: CInt
	init(descriptor: CInt) { self.id = descriptor }

	func write(buffer: [UInt8]) throws {
		let wriitten = buffer.withUnsafeBufferPointer {
			Drawin.write(fd, $0,baseAddress, $0count)
		}
		...
	}

	func close {
		Darwin.close(fd)
	}
}
```
- 어떤 경우에는 위와같이 복사가 불필요한 경우도 있다 ->  ~Copyable
- non copyable type은 class 처럼 deinit을 사용할수있음 -> 이건 언제 호출됨?(타입이 out of scope 될때 실행됨)

#### consuming
- 위의 예제에서 FileDescriptor를 ~Copyable 타입으로 바꾸고, close를 consuming으로 바꿀수 있다
- 기본적으로 swift에서 객체의 메소드는 자신을 더불어 파라미터를 borrow한다(인스턴스 메소드의 첫번째 파라미터가 자신임)
```
let file = FileDescriptor(fd: descriptor)
file.write(buffer: data)	//  여기서 self, data가 borrow되고, 호출이 끝난 뒤 소유권이 다시 돌아옴
file.close // close는 consuming 이기때문에 ownership이 종료됨
```


### C++ interop
- 생략

### Concurrency
- Task: 어디서나 실행될 수 있는 순차적인 작업의 단위
- Actor: isolated 된 상태에 대해 상호 배제적인 접근을 제공함 -> 액터 내부에 접근하기위해서는 외부에서 Task가 await 해야한다.
- task와 actor는 각기 추상적인 모형으로 제공되지만 세부적으로 다음과 같은 방식으로 구현되게 할 수 있음
- Global concurrency pool determines scheduling
ㄴ dispatch on Apple plaforms -> - task는 global concurrent pool에서 실행 가능하고, 이의 실행은 환경에 달렸다
ㄴ Single-thread cooperative queue in restricted environment -> 허나 특정 환경에서는 Apple plaforms에서 제공하는거 사용 불가능, 여기선 이렇게 구현됨
- swift 5.9에서 custom actor 구현이 제공됨 -> ex) 액터 구현의 큐를 지정할수있음
ㄴ SerialExecutor를 따르는 타입을 이용해라 -> 쓸일 없쥬?

-> 더 알고싶다면 Swift Concurrency: Behind the Scenes(2021), Beyond the basics of Structure of Concurrency(2023) 참고


### FoudationDB 
- 생략


아니 swift가 object-c 대체하는거말고
퍼포먼스, 생산성 끌어올리려고 더 복잡해지면서 다른언어대비 러닝커브는 점점 더 높아지고 장점이 뭐냐고;;
concurrency, ownership, macro 들어오면서 인터페이스 엄청 드러워짐 -> 세부 구현이 다 노출됨
기본적인 언어의 명확함과 쉬움에 더해 좀더 튜닝하고싶으면 좀더 디테일하게 건들수있는 옵션을 제공하려고하는거같은데
그렇지는 않은 느낌.. 그냥 점점 여러언어 좋은거 다 끌어모으는 짬뽕이 되고있음..





