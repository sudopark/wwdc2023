## What's new in UIKit

: https://developer.apple.com/videos/play/wwdc2023/10055/


### Key features

#### Xcode previews
```swift
// preview macro 사용

#Preview("Library") {
	let controller = LibraryController()
	return controller
}

#Preview("Memories") {
	let view = SlideshowView()
	view....
	return view
}
```

#### View controller lifecycle
- viewWillAppear 이후 viewDidAppear 이전에 호출되는 viewIsAppearing 추가됨
ㄴ vc나 view의 trait collections 업데이트
ㄴ 정확한 위치에 view를 하이라키에 추가할때 -> 사이즈도 포함하여
- back-deploys to iOS 13

##### view controller appearance
1. transaction - viewWillAppear
2. transaction - view added to hierarchy
3. Layout - view laid out by superview, trait updated
4. Layout - viewIsAppearing =>  첫번째 보이는 transtion 에 속함
5. Layout - viewWillLayoutSubviews
6. Layout - viewDidLayoutSubviews
7. transition animates...
8. Transaction - viewDidAppear

#### Trait system enhancements
- 커스텀 trait 정의 가능해짐
- Flexible trait change registration api 제공 ->  traitCollectionDidChange에서 잡을필요없음
- swiftUI와 호환 가능

#### Animation symbol images
- 새로 추가된 symbol effect api을 사용하여 심볼에 에니메이션 적용 가능
- 모든 심볼 + 커스텀한 심볼에도 적용 가능
```swift
// Bounce symbol once
imageView.addSymbolEffect(.bounce)

// add a variable color effect, which repeats
imageView.addSymbolEffect(.variableColor.iterative)

// some later, remove the effect
imageView.removeSymbolEffect(ofType: .variableColor)

// change the image, using replace effect
imageView.setSymbolImage(pauseIamge, contentTransion: .replace.offUp)
```

#### Empty states
- empty state: 보여줄 컨텐츠가 없는 경우
- UIContentUnavailableConfiguration -> empty state에 대한 합성 가능한 설명을 제공
```swift
// content 없을때 빈 메세지 출력
var config = UIContentUnavailableConfiguration.empty()
config.image = UIImage(systemName: ...)
config.text = "No Favorites"
config.secondaryText = "..."
viewController.contentUnavailableConfiguration = config

// content 없을때 로딩 출력
let config = UIContentUnavailableConfiguration.loading()
viewController.contentUnavailableConfiguration = config

// content 없을때 커스텀 뷰 적용
let config = UIHostingConfiguration {
	VStack {
		ProgressView(value: progress)
		Text("Downloading file...")
			.foregroundStyle(.secondary)
	}
}
viewController.contentUnavailableConfiguration = config
```
- 사용하는 경우(update), ~쫌 구린듯..~
```swift
override func updateContentUnavailableConfiguration(
	using state: UIContentUnavailableConfigurationState
) {
	var config: UIContentUnavailableConfiguration?
	if searchResult.isEmpty {
		config = .search()
	}
	contentUnavailableConfiguration = config
}

searchResults = backingStore.results(for: query)
setNeedsUpdateContentUnavailableConfiguration()
```

### Internationalization

#### Dynamic line-height adjustments
- uilabel에 적용된 텍스트들이 언어에 따라 읽기 쉬운 정도로 lineheight이 자동으로 조정됨 => 요거 한번 확인해봐야함

#### Improved line-breaking and hyphenation
- 중국어, 독일어, 일어, 한국어에서 라인브레이킹이 개선되었다.
- 텍스트 스타일과 언어에 대한 최적화
- 텍스트 스타일 채택 등
- 자세한 내용은 What's new with text and text interactions 참고
- 또한 텍스트 출력시에 유저가 설정한 언어와 다른 언어가 출력될수도있음 
ㄴ 이경우에 locale을 지정
```swift
let label = UILabel()
label.text = "태국어라 치자"
label.traitOverrides.typesettingLanguage = Local.Language(identifier: "th")
```

#### Retrive images by locale
- 기본적으로 현재 언어에 해당하는 이미지 로드함
```swift
imageView.image = UIImage(systemName: "character.textbox")

// locale을 이영히여 국가 지정
let locale = Locale(languageCode: .japanese)
imageView.image = UIImage(
	systemName: "character.textbox", 
	withConfiguration: UIImage.SymbolConfiguration(locale: locale)
)
```

### Improvements for iPad

#### iPad: Window dragging
- UINavigationBar 끌어서 윈도우 위치 변경 가능?
- UINavigationBar 사용 안하면 -> UIWindowSceneDragInteraction 이용
- UISplitViewController에서 윈도우 사이즈에따라 동적으로 사이드바 숨겨지거나, 위에 뿌려지거나 할 수 있음 -> 이부분 테스트 필요

#### iPad: Document support
- UIDocumentViewController -> 기본 다큐먼트앱이랑 같은 모양
- 이하 생략

#### iPad: Apple pencil
- 생략

#### iPad: keyboard scrolling
- page up, page down 등등 지원 가능해짐;;
- 이하 생략

### General enhancements

#### CollectionViews
- 퍼포먼스 개선
- 애니메이션 없이 업데이트하는 경우 더빠름
- 배치업데이트 하거나, 스냅샷으로 하거나 둘다

##### Compositional Layout
- NSCollectionLayoutDimension.uniformAcrossSiblings(estimate:)
ㄴ works with self-sizing
ㄴ 제일 큰 사이즈로 잡힘 => 이때문에 한줄에 비교군이 적은 경우 사용 권장

#### Spring animations
```swift
UIView.animate(springDuration: 0.5, debounce: 0.0) {
	circle.center.x += 100
}
// 인자 기본값 있음
UIView.animate {
	circle.center.x += 100
}
```
- 더 자세한 내용은 Animate with Springs 참고

#### Text interactions
- 텍스트 커서랑 선택 기능이 개선
- 텍스트 돋보기와 선택 핸들이 다시 디자인됨
- 시스템에서 제공하는 UI를 통해 UIInteraction 없이 커스텀 구현을 할수 있음
- UITextViewDelegate에 새로운 인터엑션 API 추가됨
- primary action이나 menu content 변경 가능
- 커스텀 레인지에 인터엑션을 위한 태그 추가 가능
-> 자세한 내용은 What's new with text and text interactions

#### Status bar
- status바 .default로 지정하면 -> 자동으로 스타일 잡아줌
- 심지어 좌우가 따로 적용될수있음
- 이제는 상황에따라 바꾸는 커스텀 코드 필요없음

#### Drag and drop
- Info.plist에 CFBundbleDocumentTypes 지정해서 앱이 드랍받을수있도록 준비해라
- 드랍 받으면 UIScene의 delegate callback이 호출되며 열릴꺼다 -> url 처리하듯이 비슷하게 처리해라

#### HDR Images
- 생략

#### Page control
- uipageControl에 타이머 플러그인 붙었음
```swift
// 10초마다 페이지 바꾸는 경우
// 이러면 현재 페이지에서 10초간 페이지 전환 프로그래스 보임
let timeProgress = UIPageControlTimerProgress(perferredDuration: 10)
pageControl.progress = timeProgress
timeProgress.resumeTimer()

// 혹은 커스텀하게 돌림
let progress = UIPageControlProgress()
pageControl.progress = progress

myTimer.addPeriodicTimeObserver { timer in
	progress.currentProgress = Float(timer.seconds / timer.duration)
}

```

#### Palette menus
- menu에 컬러 팔레트 선택 메뉴 추가됨
- 심볼로도 적용 가능
