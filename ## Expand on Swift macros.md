## Expand on Swift macros

: https://developer.apple.com/videos/play/wwdc2023/10167/


### Why macro
- codable 디폴트 구현처럼 이미 매크로처럼 동작하는 기능들이 몇개 있다
- 사용자들이 proposal을 통해 추가할수는 있지만 이는 scalable 하지 않다.  

#### Macro
- 컴파일러 변경 없이 swift 코드를 확장한다.
- 지겨운 반복작업을 없애고 -> 이를 package로 공유도 가능함


### Design Philosophy
: 4가지의 목표
1. 사용처가 분명함
- Freestanding macro: 코드 expression이나 선언부에도 쓰일수있음, #으로 시작
```swift
return #unwrap(icon, message: "error message")
```
- Attached macros: 선언부에 추가되며 @으로 시작
```swift
@AddCompletonHandler func sendRequest() async throws -> Response
```
- swift는 이미 컴파일러의 특수 동작을 위헤 #, @ 사인을 사용중 -> 매크로는 이를 확장
2. Complete, type-checked, validated
3. Inserted in pretictable ways
- 매크로는 현재 스코프 내에서만 추가 가능?
```swift
func foo() {

	start()

	// 이 매크로가 뭔지 세부동작을 모르더라도, 기존 플로우가 변경되는 것은 아님
	#someMacro()

	finish()
}
```
4. Macros are not magic
- 매크로는 단순히 코드만 추가시키고 이를 xcode에서 볼수있다. -> expand하고 디버깅도 가능
- closed 라이브러리에서 임포트한 macro도 expand 가능, macro 자체에 유닛테스트 추가 가능(추천)


### Transalation Model
1. swift 컴파일러가 코드내 매크로를 분리하고, 해당 매크로의 구현이 존재하는 특정 컴파일러 플러그인에 이를 보냄
2. 플러그인은 secure sandbox안에서 분리되 별도 프로세스로 실행되고 여기에 매크로 작성자의 swift 코드가 있음
3. 매크로를 처리하여 생산된 코드 조각 "expansion"을 반환
4. swift 컴파일러는 이를 코드 내에 삽입

- 그러면 컴파일러는 swift 코드 내에서 매크로가 존재하는지 어떻게 알까? -> macro 선언부
- macro declaration은 매크로 API를 제공
```swift
@freestanding(expression) // macro role
macro some<T>(_ expr: T) -> (T, String)
```

### Macro Roles
- 어디에 쓰일것이고, 어떤 타입의 코드가 expand 될것이며, 이 expansion이 코드의 어느 부분에 추가될것인지 role이 결정함

#### freestanding
1. @freestanding(expression) - 값을 반환하는 코드 조각을 생성
- expression은 실행되고 결과를 만들어내는 코드조각을 의미
```swift
// = 우변이 expression, 이는 재귀적인 구조를 지니고 x + width 또한 expression
let pixels = (x + width) * (y + height)
```
```swift
// unwrap 예제
// Force-unwrap the optional value passed to 'expr'
//  - parameter message: Failure message, followed by 'expr' in single quotes
@freestanding(expression)
macro unwrap<Wrapped>(_ expr: Wrapped?, message: String) -> Wrapped

// 사용처 및 expansion

let image = #unwrap(downloadedImage, message: "was already checked")
   // expansion
   { [downloadedImage] in
   		guard let downloedImage 
   		else {
   			preconditionFailure(
   				"Unexpectedly found nil: 'downloadedImage" " + "was already checked",
   				file: "main/ImageLoader.swift", 
   				line: 42
   			)
   		}
   		return downloadedImage
   }
   // expansion end

```
2. @freestanding(declaration) - 하나 이상의 선언을 생성
- 함수, 변수, 타입을 expand 할 수 있음

#### attached
- attached 매크로는 선언에 붙고, 선언부에 접근 가능
1. @attached(peer) - 새로운 선언을 타겟에 추가하고 적용
- 어느 선언부에나 붙을 수 있음
- 메소드나, 프로퍼티에 이를 추가하면 -> 타입에 새로운 멤버가 생기고
- top-level 펑션이나 타입에 추가하면 -> 새로운 top-level declaration이 만들어짐
2. @attached(accessor) - accessor를 프로퍼티로 추가
- varidable이나 subscript에 붙어 get, set, willSet, didSet 들을 추가함
```swift
@attached(accessor)
macro DictionaryStorage(key: String? = nil)
```
3. @attached(memberAttribute) - type이나 extension에 attribute 선언을 추가?
```swift
@attached(memberAttribute)
@attached(accessor)
macro DictionaryStorage(key: String? = nil)

struct Person: DictionaryRepresentable  {

	// dict이랑 init는 매크로에서 스킵하도록 구현되어있음
	var dict: [String: Any]
	init(dict: [String: Any])  {
		self.dict = dict
	}

	// @DictionaryStorage
	var name: String
	// @DictionaryStorage
	var height: Measuremement<UnitLength>
	@DictionaryStorage(key: "birth_date") var birthDate: Date?
}
```
- macro role은 합성 가능
4. @attached(member) - type/extension이 적용된 부분에 새로운 선언을 추가?
```swift
@attached(member, names: named(dict), named(init(dict:)))
@attached(memberAttribute)
@attached(accessor)
macro DictionaryStorage(key: String? = nil)

@DictionaryStorage struct Person: DictionaryRepresenable {
	// 생성자랑 dict 은 매크로가 expand 해줌
	var name: String
	var height: Measuremement<UnitLength>
	@DictionaryStorage(key: "birth_date") var birthDate: Date?
}

```
5. @attached(conformance) - type/extension이 적용된 부분에 conformance를 추가
```swift
@attached(conformance)
@attached(member, names: named(dict), named(init(dict:)))
@attached(memberAttribute)
@attached(accessor)
macro DictionaryStorage(key: String? = nil)

@DictionaryStorage struct Person {
	// 생성자랑 dict 은 매크로가 expand 해줌
	var name: String
	var height: Measuremement<UnitLength>
	@DictionaryStorage(key: "birth_date") var birthDate: Date?
}
```
과도하게 코드를 압축하는게 좋은건지 모르겠네..


### Macro Implementation
```swift
// = 다음에 다른 매크로가옴
@freestanding(expression)
macro foo<T>(_ expr: T) -> (T, String) = <Implementation goes here>

// ex1 내가 정의한 다른 매크로
@freestanding(expression)
macro foo<T>(_ expr: T) -> (T, String) = #fooWithPrefix(expr, prefix: "some")

// ex2 하지만 대부분의 경우 external macro가 이어짐
@freestanding(expression)
macro foo<T>(_ expr: T) -> (T, String) = #externalMacro(
	module: "MyLibMacros", 	// 어떤 플러그인에서
	type: "FooMacro"		// 어떤 매크로 타입을 사용할것인지
)
```
- external macro는 컴파일러 플러그인에 따라 구현됨
- DictionaryStorage 실제 구현 예제
```swift
import SwiftSyntax 			// swift 코드 파싱, 검사, 조작, 생성 하는것을 도와줌
import SwiftSyntaxMacros	// 
import SwiftSyntaxBuilder

struct DictionaryStorageMacro: memberMacro {

	// 여기서 파라미터는 memberMacro를 구현하기 위한 재료
	static func expansion(
		of attribute: AttributeSyntax, 
		providingMemberOf decleration: some DeclGroupSyntax, 
		in conext: some MacroExpansionContext
	) throws -> [DeclSyntax] {
		// return된 [DeclSyntax]가 소스코드에 추가될것임
		return [
			"init(dict: [String: Any]) { self.dict = dict }", 
			"var dict: [String: Any]"	// 여기서 스트링으로 되어있지만 실제로는 DeclSyntax string literal임
		]
	}
}
```
1. SwiftSyntax는 트리 구조로 소스코드를 나타낸다 -> StructDeclSyntax
```swift
@DictionaryStorage
struct Person {
	var name: String
	var height: Measurement<UnitLength>
	@DictionaryStorage(key: "birth_date") var birthdate: Date?
}

// DictionaryStorage로 보면 이 소스 코드는
- attributes -> AttributeListSyntax => @DictionaryStorage
- modifier -> nil
- structKeyword -> TokenSyntax.keyword "struct" => struct
- identifier -> TokenSyntax.identifier "Person" => Person
- memberBlock -> MemberDeclBlockSyntax => 나머지 부분
```
- 위 property들의 일부 syntax 노드는 "token"이라 불리며 소스코드 파일의 특정 텍스트 조각을 나타낸다.
- 위의 예제에서 AttributeListSyntax, MemberDeclBlockSyntax는 토큰이 아니다 -> 더 나눠질수있음
-> 요거는 내용이 복잡해서 다음 영상을 보던가 SwiftSyntax 다큐먼트를 봐라

2. SwiftSyntaxMacros -> 매크로를 구현하기위한 타입이나 프로토콜을 제공
3. SwiftSyntaxBuilder -> syntax 트리를 만들기 위한 API를 제공
-> 왠만해선 이것들 이용해서 매크로 구현하셈 ㅇㅇ

- macro룰은 각자 프로토콜을 나타냄
- 위의 예제에서 MemberMacro를 구현했고 나머지들도 해줘야함
```swiift
extension DictionaryStorageMacro: ConformanceMacro, MemberAttributeMacro, AccessorMacro { ... }
```
- 위 매크로 룰을 어겼을때 컴파일 에러는 나지만 에러 메세지 구분이 어렵다  -> 이를 위한 구현을 수정
```swift
static func expansion(
	of attribute: AttributeSyntax, 
	providingMemberOf decleration: some DeclGroupSyntax,  // struct, class, protocol, actor와 같은 타입 정보가 들어있음
	in conext: some MacroExpansionContext // 컴파일러와 상호작용할때 쓰임, 에러메제지나 워닝 표시
) throws -> [DeclSyntax] {
	
	// is macro attached to struct?
	guardd declaration.is(StructDeclSyntax.self) else {
		// throw NotStructError() -> 이렇게 에러를 throw할수있지만 큰 정보를 주지는 못한다.
		let structError = Diagnostic(
			node: attribute, 	// location of error
			message: MyLibDiagnostic.notAStruct 	// error message/severity
		)
		context.diagnose(structError)
		return []	// 빈값을 반환하면 추가되는 코드 없음
	}
	return [
		"init(dict: [String: Any]) { self.dict = dict }", 
		"var dict: [String: Any]"	// 여기서 스트링으로 되어있지만 실제로는 DeclSyntax string literal임
	]
}

enum MyLibDiagnostic: String, DiagnosticMessage {
	case notAStruct

	var severity: DiagnosticSeverity { return .error }

	var message: String {
		switch self {
			case .notAStruct: return "'@DictionaryStorage' can only be applied to a 'Struct'"
		}
	}

	var diagnosticID: MessageID {
		MessageID(domain: "MyLibMacros", id: rawValue)
	}
}
```


#### Building syntax trees
- syntax initalizer and methods
- swiftUI styles syntax builders
- Literals with interpolations

```swift
// implementing the #unwrap expression macro
static func makeGuardStmt(
	wrapped: TokenSyntax,
	originalWrapped: ExprSyntax,
	message: ExprSyntax, 
	in context: some MacroExpansionContext
) -> StmtSyntax {

	let messagePrefix = "Unexpectedly found nil: '\(originalWrapped.description)'"
	let originalLoc = context.location(of: originalWrapped)!

	return """
	guard let \(wrapped) 
	else {
		preconditionFailure(
			\(literal: messagePrefix) + \(message),
			file: \(originalLoc.file), 
			line: \(originalLoc.line)
		)
	}
	"""
}
```

### Writing correct macros
- name conflict -> expand되기 이전 구문에 존재하는 이름들과 출동날수있음
```swift
// 이를 미연에 방지하기위해
let captureVar = context.makeUniqueName() // make a tokenSyntax with a name that's guranteed unique
return """
{
	[\(captureVar) = \(originalWrapped)] in
		\(makeGuardStmt(wrapped: captureVar, ...))
		\(makeReturnStmt(wrapped: captureVar))
}
"""
```

#### name specifiers
1. overloaded -> attached 된 선언부와 같은 이름으로 맨들어라
2. prefixed(<some prefix>) -> 기존 선언부 이름 앞에 prefix 추가
3. suffx(<some suffix>)
4. name(<some name>) -> 이 이름으로 맨들어라
5. arbitrary -> 다른 규칙들이 표현하지 못하는 다른 이름으로 맨들어라 -> 컴파일러 속도에는 별로 안좋음?

### Don't use outside information
- 컴파일러가 제공하는 정보만 이용해라
- 매크로는 파일 시스템이나 네트워크에 접근할수없다
- sandbox 정보 접근하는거 비추(current time, process ID or rand number)
- expansion등을 글로벌 변수로 사용하거나 하지 말어라잉

### Testing your macros
- 의도한바로 expand 하는지를 테스트
```swift

import XCTest
import MyLibMacros
import SwiftSyntaxMacrosTestSupport	// import test helpers

final class MyLibTests: XCTestCase {
	func testMacro() {
		assertMacroExpansion(
			"@DictionaryStorage var name: String", 	// 매크로가 생산하는 expression
			expandedSource: """						// 매크로가 expand 된 경우
			var name: String {
				get { ... }
				set { ... }
			}
			""", 
			macros: ["DictionaryStorage": DictionaryMacro.self]
		)
	}
}

```
