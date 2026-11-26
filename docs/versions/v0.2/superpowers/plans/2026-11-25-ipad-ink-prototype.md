# Me-mento iPad 필기 프로토타입 구현 계획서

문서 v1.1 · 2026-11-26 · 제품 V0(M0~M2)
> **구현 에이전트:** 작업별로 superpowers:subagent-driven-development 또는 superpowers:executing-plans를 사용한다. 각 단계의 체크박스에 실행 결과를 기록한다.

**목표:** 실제 iPad에서 한 페이지를 쓰고 지운 뒤 앱을 종료·재실행하여 마지막 저장 결과에서 이어 쓴다.

**구조:** SwiftUI 편집기 안에 PencilKit의 `PKCanvasView`를 둔다. 캔버스가 입력·스크롤·확대와 편집 세션의 실행 취소를 담당하고, 메인 액터의 저장 조정자가 캔버스에서 받은 스냅샷을 직렬화하여 기기 내 파일에 저장한다.

**기술:** Xcode, Swift, SwiftUI, PencilKit, Foundation, XCTest. 외부 패키지 없이 시작한다.

**근거:** [구현 명세 INK-01/02](../../implementation-spec.md#3-편집기와-로컬-저장--ink-01--ink-02--doc-01--map-01) · [출범 계획 M0~M2](../../project-kickoff.md#3-개발-단계와-전환-조건) · [UX-01](../../ux-storyboard.md#ux-01--필기하고-다시-열기) · [개발 로드맵](../../development-roadmap.md) · [문서 개정 이력](../../README.md)

## 전체 제약

- V0의 M1~M2 내부 빌드는 한 페이지 편집기로 시작한다. V1의 홈·최근 노트와 문서 가져오기는 M3, 마인드맵은 M4, 로그인과 공개·복사는 M5에서 연결한다. V0는 UX-01의 필기·저장·다시 열기를 검증한다.
- 손가락 필기 기본값은 꺼짐이다. 꺼짐에서는 Pencil로 쓰고 지우며 손가락으로 이동·확대한다. 켜짐에서는 손가락 하나로 쓰고 지우며 두 손가락으로 이동·확대한다. Pencil도 계속 입력할 수 있다. 마지막 선택은 재실행 후 유지한다.
- 실행 취소·다시 실행은 현재 편집 세션에 적용한다. 재실행 후에는 저장된 결과를 복원한다.
- 정상 저장본을 복원하거나 새 페이지임을 확인한 뒤에만 필기를 시작한다. 저장 실패 시 편집 중인 결과와 이전 정상 파일을 유지하고 재시도를 제공한다.
- 최소 iPadOS는 제품 명세에서 미정이다. 프로토타입 배포 대상은 첫 실기기의 iPadOS에서 실행할 수 있는 값으로 설정하고, 기기 모델·iPadOS·Xcode 버전과 함께 기록한다. 제품 전체의 지원 하한은 M3 전에 정한다.
- 빌드·시뮬레이터 검증은 macOS와 Xcode에서, Pencil·손바닥·터치 제스처 확인은 실제 iPad에서 수행한다. 실행하지 않은 항목은 **미실행**으로 기록한다.

## 검토에서 먼저 볼 다섯 가지

| 상황 | 기대 결과 | 검증 |
|---|---|---|
| 페이지 파일이 처음부터 없음 | 쓰기 가능한 빈 페이지가 열린다. | 작업 2 · S-01 |
| 기존 파일이 손상되거나 지원하지 않는 버전임 | 로드 오류를 표시하고 파일을 보존한다. | 작업 2 · S-02, 작업 4 · D-07 |
| 최종 획을 지워 페이지가 비어 있음 | 종료 후에도 빈 페이지로 열린다. | 작업 2 · S-03, 작업 4 · D-05 |
| 저장 중 새 획이 들어옴 | 이전 저장이 완료돼도 최신 변경은 저장 중으로 남고, 마지막 변경까지 기록된다. | 작업 3 · C-01 |
| 저장 실패 후 편집을 한 번 더 함 | 현재 편집은 남으며 재시도는 실패 시점의 값보다 최신 편집을 저장한다. | 작업 3 · C-02, 작업 4 · D-06 |

## 파일과 실행 명령

경로는 저장소 루트 기준이다. 아래 앱·테스트 프로젝트는 이 계획을 실행할 때 생성한다.

| 파일 | 역할 |
|---|---|
| `ios/MeMento.xcodeproj/project.pbxproj`, 공유 스킴 `MeMento` | iPad 앱과 앱 호스트 단위 테스트. 기기 서명 설정은 각 개발 환경에서 구성한다. |
| `ios/MeMento/MeMentoApp.swift` | 첫 화면, 저장 위치, 편집기와 저장소의 연결 |
| `ios/MeMento/InkEditorView.swift` | 캔버스와 도구, 손가락 필기 토글, 저장 상태와 재시도, 앱 활성 상태 |
| `ios/MeMento/PencilCanvasView.swift` | `UIViewRepresentable`과 `PKCanvasView`의 입력·이벤트 연결 |
| `ios/MeMento/InkPageStore.swift` | 버전이 있는 단일 페이지 파일의 읽기·검증·원자적 교체 |
| `ios/MeMento/InkSaveCoordinator.swift` | 초기 로드, 저장 순서, 상태, 실패 재시도 |
| `ios/MeMentoTests/InkPageStoreTests.swift` | 파일 저장·로드의 단위 테스트 |
| `ios/MeMentoTests/InkSaveCoordinatorTests.swift` | 저장 진행·실패를 제어하는 편집 세션 테스트 |

첫 작업에서 Xcode 프로젝트를 만든 뒤 저장소 루트에서 실행한다. `SIM_ID`에는 같은 Mac에 설치된 iPad 시뮬레이터의 실제 UUID를 쓴다. 실기기 설치는 Xcode에서 기기와 서명 팀을 선택해 Run으로 확인한다.

```sh
xcodebuild -version
xcodebuild -project ios/MeMento.xcodeproj -scheme MeMento -showdestinations
xcodebuild -project ios/MeMento.xcodeproj -scheme MeMento \
  -destination "platform=iOS Simulator,id=${SIM_ID:?iPad 시뮬레이터 UUID를 지정하세요}" build
xcodebuild -project ios/MeMento.xcodeproj -scheme MeMento \
  -destination "platform=iOS Simulator,id=${SIM_ID:?iPad 시뮬레이터 UUID를 지정하세요}" test
```

특정 테스트 실행에는 마지막 명령에 `-only-testing:MeMentoTests/InkPageStoreTests` 또는 `-only-testing:MeMentoTests/InkSaveCoordinatorTests`를 추가한다. 빌드는 `BUILD SUCCEEDED`, 테스트는 예정된 테스트가 한 개 이상 수행되고 실패 0개와 `TEST SUCCEEDED`가 확인되어야 한다.

## 작업 1 · iPad 한 화면 필기 실험 (M0~M1)

**파일** · 프로젝트와 공유 스킴, `MeMentoApp.swift`, `InkEditorView.swift`, `PencilCanvasView.swift`.

**인터페이스** · 편집기는 `InkTool.pen / .eraser`, `allowsFingerDrawing: Bool`을 전달한다. 캔버스는 `PencilCanvasView(tool:allowsFingerDrawing:onDrawingChanged:)`로 이를 받고, 그리기 변경을 `onDrawingChanged: (Data) -> Void`로 내보낸다. `Data`는 `PKDrawing.dataRepresentation()`이다.

- [ ] Xcode에서 iPad용 SwiftUI 앱 `MeMento`와 XCTest 대상 `MeMentoTests`를 만든다. 공유 스킴에 앱과 테스트를 연결한다. 앱을 설치해 빈 화면을 열고 Xcode·기기·iPadOS를 기록한다.
- [ ] 펜과 지우개를 제공하는 한 화면을 만든다. `PKCanvasView`를 화면 갱신 사이에 유지하고, 도구나 입력 모드를 바꿀 때 기존 Drawing을 재설정하지 않는다.
- [ ] 손가락 모드에 맞춰 `.pencilOnly`와 `.anyInput`을 사용한다. 페이지 크기는 회전과 확대에도 일정한 문서 좌표로 유지한다. 스크롤·핀치는 캔버스와 경쟁하는 별도 스크롤 뷰 없이 검증한다. 토글 값은 `@AppStorage("ink.allowsFingerDrawing")`에 기본값 `false`로 저장한다.
- [ ] 시뮬레이터 빌드를 확인한다. 실제 iPad에서 D-01~D-03을 실행해 M1을 판정한다. 제스처가 충돌하면 감지 조건을 조정하고 세 항목을 다시 실행한다.
- [ ] 작업 단위로 커밋한다. 메시지 예시 `iPad 한 페이지 필기와 입력 모드 추가`.

## 작업 2 · 한 페이지 파일과 복원 계약 (M2)

**파일** · `InkPageStore.swift`, `InkPageStoreTests.swift`.

**인터페이스** · `protocol InkPageStoring: Sendable`은 `func load() async throws -> Data?`와 `func save(_ drawingData: Data) async throws`를 제공한다. `InkPageStore`는 이 계약을 구현하고 `init(fileURL: URL, writeAtomically: @escaping @Sendable (Data, URL) throws -> Void)`로 파일 위치와 쓰기 동작을 받는다. 기본 쓰기는 `Data.write(to:options: .atomic)`을 사용한다.

- [ ] XCTest에서 임시 디렉터리를 사용해 표의 S-01~S-04를 작성한다. 테스트에서 호출할 저장 계약을 선언한 뒤 실행해, 필요한 저장 동작이 아직 없어 실패함을 확인한다.

| ID · 테스트 이름 | 입력과 단언 |
|---|---|
| S-01 · `testMissingFileReturnsNil` | 존재하지 않는 페이지 경로에서 `load() == nil`을 확인한다. |
| S-02 · `testUnreadablePayloadLeavesExistingFileUntouched` | 잘못된 형식, 다른 버전, 유효한 봉투 안의 잘못된 Drawing을 각각 준비한다. 로드가 오류를 반환하고 파일 바이트가 유지된다. |
| S-03 · `testEmptyDrawingReplacesPreviousInk` | 유효한 비어 있지 않은 Drawing을 쓴 뒤 빈 `PKDrawing`을 저장한다. 새 저장소에서 로드한 Drawing의 획 수는 0이며 결과는 `nil`이 아니다. |
| S-04 · `testFailedReplacementKeepsLastSavedDrawing` | 정상 Drawing A를 저장한다. 교체 단계에서 오류를 내는 쓰기 동작으로 Drawing B를 저장하면 오류가 반환되고 새 저장소에서 A가 로드된다. |

- [ ] 저장 봉투는 버전 1과 PencilKit Drawing의 `Data`를 가진다. 로드 시 봉투 버전과 Drawing 디코딩을 확인한다. 파일 없음만 `nil`로 처리한다. 오류가 난 파일은 유지한다. 저장 경로의 부모를 준비하고 기존 파일을 원자적으로 교체한다.
- [ ] 앱 기본 경로는 Foundation이 구한 Application Support 하위 `MeMento/prototype-page.ink`로 정한다. 테스트는 운영 파일과 분리된 임시 URL을 쓴다.
- [ ] `InkPageStoreTests`를 다시 실행해 4개가 통과하는지 확인한다. 한글 메시지 `한 페이지 필기 파일 저장과 복원 추가`로 커밋한다.

## 작업 3 · 자동 저장 순서, 실패와 재시도 (M2)

**파일** · `InkSaveCoordinator.swift`, `InkSaveCoordinatorTests.swift`.

**인터페이스** · `@MainActor final class InkSaveCoordinator: ObservableObject`는 `init(store: any InkPageStoring)`, `func load() async`, `func drawingDidChange(_ data: Data)`, `func retrySave()`, `func flushPending()`을 제공한다. `private(set) var drawingData: Data?`와 `private(set) var phase: InkSavePhase`를 읽을 수 있게 한다. `InkSavePhase`는 `.loading, .ready, .saving, .saved, .loadFailed, .saveFailed`를 구분한다. 화면을 갱신하는 값은 관찰할 수 있어야 한다.

- [ ] 저장 호출 완료 시점과 오류를 테스트에서 지정할 수 있는 가짜 저장소를 만든다. 실제 시간 지연에 의존하지 않고 C-01~C-03을 작성한다. 조정자의 서명과 상태 선언을 만든 뒤 테스트를 실행해 저장 동작이 아직 부족해 실패함을 확인한다.

| ID · 테스트 이름 | 입력과 단언 |
|---|---|
| C-01 · `testNewEditDuringWriteRemainsPendingUntilSaved` | 로드 후 A를 쓰는 동안 완료를 보류한다. B와 C 편집 후 A를 성공시키면 완료 표시가 나오지 않고 C가 저장된다. C 성공 뒤에만 `.saved`이며 최종 값은 C다. |
| C-02 · `testRetryWritesLatestEditAndKeepsFailedDrawingVisible` | A를 로드하고 B 저장을 실패시킨다. 화면 데이터가 B이고 디스크가 A인지 확인한다. C를 추가한 뒤 재시도하면 C가 한 번 기록되고 `.saved`가 된다. |
| C-03 · `testLoadFailureDoesNotWriteBlankPage` | 로드를 실패시킨다. `.loadFailed`를 확인하고 변경 입력과 저장 요청을 보내도 쓰기 호출이 0회인지 확인한다. 다시 읽기에 성공한 후에만 편집을 시작한다. |

- [ ] 초기 로드 완료까지 변경 저장을 막는다. 최신 입력을 기록하고 동시에 쓰기 하나만 실행한다. 새 변경이 쌓이면 이전 쓰기 뒤 가장 최신 값을 기록한다. 오래된 작업의 성공으로 최신 변경을 완료 처리하지 않는다.
- [ ] 쓰기에 실패하면 현재 메모리 데이터를 유지하고 `.saveFailed`를 표시한다. 실패 상태에서 추가로 쓰면 편집만 반영한다. `retrySave()`가 최신값을 저장하고, `flushPending()`은 활성 저장 또는 저장 가능한 대기 편집을 이어 처리한다. `flushPending()`은 실패 상태에서 자동 반복 쓰기를 일으키지 않는다.
- [ ] `InkSaveCoordinatorTests`와 전체 XCTest를 실행한다. 한글 메시지 `필기 자동 저장과 실패 재시도 추가`로 커밋한다.

## 작업 4 · 캔버스와 저장 연결, 실행 취소, 실기기 인수 (M2)

**파일** · `MeMentoApp.swift`, `InkEditorView.swift`, `PencilCanvasView.swift`. 실기기 결과는 아래 검증표에 기록한다.

**인터페이스** · `PencilCanvasView`에 `initialDrawingData: Data?`, `onUndoManagerChanged: (UndoManager?) -> Void`, `onUndoStateChanged: (Bool, Bool) -> Void`를 추가한다. 페이지를 준비할 때 초기값을 한 번만 반영한다. 캔버스는 실제 사용하는 UndoManager를 편집기에 전달하고 캔버스가 제거되면 nil을 전달한다. 편집기 버튼은 해당 `UndoManager`의 `undo()`와 `redo()`를 호출한다. 화면은 `InkSaveCoordinator.phase`에 따라 상태와 버튼을 표시한다.

- [ ] 앱에서 실제 저장소와 조정자를 연결한다. 첫 로드 중 상태를 표시하고, `.loadFailed`에는 읽기 재시도를 제공한다. 로드에 성공하면 빈 페이지 또는 복원한 Drawing을 초기값으로 전달한다. 초기 렌더링 자체가 새 편집이나 실행 취소 동작으로 잡히지 않게 한다.
- [ ] 캔버스의 그리기·지우기·실행 취소·다시 실행 결과를 `drawingDidChange`에 연결한다. 캔버스의 실행 취소 가능 여부로 버튼 상태를 바꾼다. 재실행 시 이력은 새로 시작한다.
- [ ] 편집 중에는 `저장 중…`, 성공 뒤에는 `저장됨`, 실패하면 `아직 저장하지 못했어요.`와 `다시 시도`를 표시한다. 실패 안내가 캔버스를 가리지 않게 한다. 앱의 비활성화와 백그라운드 진입을 관찰해 대기 저장을 가능한 범위에서 처리한다. 같은 시점에 들어온 두 신호가 중복 쓰기를 만들지 않게 한다.
- [ ] Debug 빌드에서만 `--ink-test-store`로 별도 시험용 경로를 선택한다. 이 인자가 있을 때만 `--ink-fail-next-write`로 다음 쓰기 한 번을 실패시키고, `--ink-seed-corrupt`로 손상 검증 전용 경로를 선택해 파일이 없을 때만 손상 자료를 준비하게 한다. 실제 저장 경로에는 이 인자를 적용하지 않는다.
- [ ] 전체 XCTest를 실행하고 D-04~D-07을 실제 iPad에서 수행한다. 변경 이후 D-01~D-03도 재확인한다. 실제 결과와 재현 조건을 적는다. 한글 메시지 `필기 화면에 저장 복원과 실행 취소 연결`로 커밋한다.

## 실기기 검증표

**기록할 환경** · 빌드 커밋, 검증 날짜, Xcode, iPad 모델·iPadOS, Apple Pencil 모델. 결과란에는 **통과 / 실패 / 미실행**과 관찰 내용을 적는다. 테이블은 실행 전 기준이다.

| ID | 기기에서 할 행동 | 통과 기준 | 결과 |
|---|---|---|---|
| D-01 | 손가락 모드를 끄고 손바닥을 올려 Pencil로 쓰고 지운다. 손가락으로 페이지를 밀고 두 손가락으로 확대한다. | 의도한 곳에만 획이 생긴다. 손가락으로 이동·확대할 때 새 획이 생기지 않는다. | 미실행 |
| D-02 | 손가락 모드를 켜고 한 손가락으로 쓰고 지운다. 두 손가락으로 이동·확대하고 Pencil로도 쓴다. | 두 손가락 제스처가 획으로 처리되지 않고 Pencil 입력을 계속 쓸 수 있다. | 미실행 |
| D-03 | 손가락 모드를 켜고 앱을 종료·재실행한다. 한 획으로 모드가 유지됐는지 확인한다. | 선택한 모드와 동작이 일치한다. | 미실행 |
| D-04 | A와 B를 쓰고 B를 지운 뒤 실행 취소·다시 실행한다. `저장됨`에서 앱을 강제 종료하고 다시 연다. | A만 보이고 이어 쓸 수 있다. 새 세션의 실행 취소 기록은 비어 있다. | 미실행 |
| D-05 | 모든 획을 지운 뒤 `저장됨`을 확인하고 앱을 다시 연다. | 빈 페이지가 복원된다. 이전 획이 되돌아오지 않는다. | 미실행 |
| D-06 | `--ink-test-store --ink-fail-next-write`로 실행해 쓰기 실패를 일으킨다. 실패 상태에서 새 획을 추가하고 `다시 시도`한 다음 `--ink-test-store`만 켜고 앱을 다시 연다. | 실패 안내 중 두 편집이 화면에 남는다. 재시도 성공 후 두 편집이 재실행에서 복원된다. | 미실행 |
| D-07 | `--ink-test-store --ink-seed-corrupt`로 앱을 연다. 로드 재시도를 누른다. | 읽기 오류와 재시도가 보이고, 준비한 파일이 교체되지 않으며 빈 화면을 정상 로드로 취급하지 않는다. | 미실행 |

**단계 판정** · M1은 D-01~D-03, M2는 전체 XCTest와 D-01~D-07의 실제 결과를 기준으로 한다. Xcode·실기기 환경이 준비되지 않았으면 해당 항목을 미실행으로 남긴다. 사용자가 저장 완료로 확인한 내용이 사라지거나 의도하지 않은 획이 남는 경우 원인을 확인하고 관련 시나리오를 다시 실행한 뒤 M3 진입을 판정한다.
