---
title: Flutter Riverpod 핵심 개념 정리 (Provider, Ref, Notifier, DI)
date: 2026-11-12 09:00:00 +0900
categories: [Flutter, Riverpod]
tags: [flutter, riverpod, state-management]
toc: true
---

Riverpod의 핵심 용어, UI에서 상태를 구독하는 위젯, 의존성 주입(DI) 원리, 수식어(`.autoDispose`, `.family`)를 정리하고 카운터 예제

## 1. Riverpod의 핵심 용어

### 1-1. ProviderScope (상태 저장소 루트)

- **역할**: 앱 전체에서 사용되는 모든 Provider의 실제 상태(데이터)가 저장되고 관리되는 최상위 컨테이너
- **특징**
  - `runApp(ProviderScope(child: MyApp()))` 형태로 앱 최상단을 감싸야 한다.
  - Provider는 전역 변수처럼 선언하지만, 실제 상태는 `ProviderScope` 내부의 격리된 공간에 저장된다.
  - 그래서 테스트할 때마다 상태를 초기화하기 쉽다.

```dart
void main() {
  runApp(const ProviderScope(child: MyApp()));
}
```

### 1-2. Provider (상태 및 객체의 설계도)

- **역할**: 데이터나 객체(Repository, UseCase, Notifier 등)를 생성하고 노출하는 선언적 정의
- **특징**: 아래처럼 전역(Global) 변수로 선언하지만, 실제 값은 전역에 있지 않고 `ProviderScope`에 저장되므로 안전하다.

```dart
final countProvider = Provider<int>((ref) => 0);
```

**주요 종류**

| Provider | 용도 |
|----------|------|
| `Provider` | 변하지 않는 객체나 단순 계산 값 제공 (Repository, UseCase 등 DI용) |
| `NotifierProvider` | 동기적으로 변하는 상태와 그 상태를 변경하는 로직(Notifier)을 함께 제공. 구 `StateNotifierProvider`의 최신 대체제 |
| `AsyncNotifierProvider` | API 호출 등 비동기로 로드되고 변경되는 상태(`AsyncValue`)를 다룰 때 사용 |
| `FutureProvider` / `StreamProvider` | 상태 변경 메서드 없이 단순 비동기 데이터(단발성 API 조회, 소켓 스트림 등)만 읽어올 때 사용 |

### 1-3. Ref (Provider 간의 연결 고리)

- **역할**: Provider 내부나 위젯에서 다른 Provider에 접근할 수 있게 해주는 참조 객체
- **종류**: Provider 내부에서 쓰는 `Ref`, 위젯에서 쓰는 `WidgetRef`

**핵심 메서드 3가지**

| 메서드 | 동작 | 주로 쓰는 곳 |
|--------|------|--------------|
| `ref.watch(provider)` | 값을 읽고, 값이 바뀔 때마다 해당 위젯/Provider를 다시 빌드 | UI를 그릴 때, 다른 Provider의 변화에 반응할 때 |
| `ref.read(provider)` | 값이나 Notifier 인스턴스를 **한 번만** 읽음. 값이 바뀌어도 리빌드 없음 | 버튼 클릭(`onPressed`) 같은 이벤트 콜백 안에서 메서드 호출 |
| `ref.listen(provider, (previous, next) { ... })` | 상태 변경을 감지하되 화면은 다시 그리지 않고 부수 효과(Side Effect)만 실행 | 에러 시 스낵바 띄우기, 로그인 성공 시 페이지 이동 |

### 1-4. Notifier & AsyncNotifier (상태 관리 본체)

- **역할**: 실제 상태(`state`)를 보유하고, 그 상태를 변경하는 비즈니스 메서드를 모아둔 클래스
- **특징**
  - 내부에 `state` 프로퍼티를 기본으로 가진다.
  - `state = 새로운값` 형태로 **불변(Immutable) 객체를 갈아끼우면**, 이 Provider를 `watch`하는 모든 곳에 자동으로 변경이 통지된다.

## 2. UI에서 상태를 구독하는 위젯 (Consumer 계열)

Riverpod에서는 일반 `StatelessWidget`, `StatefulWidget` 대신 `WidgetRef ref`를 제공해 주는 전용 위젯을 사용한다.

### 2-1. ConsumerWidget (StatelessWidget 대체)

가장 기본이 되는 위젯. `build` 메서드에 `WidgetRef ref`가 추가된다.

```dart
class MyPage extends ConsumerWidget {
  const MyPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(countProvider); // ref를 통해 상태 구독
    return Text('$count');
  }
}
```

### 2-2. ConsumerStatefulWidget & ConsumerState (StatefulWidget 대체)

`TextEditingController`, `AnimationController`처럼 위젯 자체의 로컬 생명주기(`initState`, `dispose`)가 필요할 때 사용한다. `State` 클래스 내부에서 `ref`를 바로 꺼내 쓸 수 있어서 `build` 파라미터에 `ref`를 적지 않아도 된다.

```dart
class MyFormPage extends ConsumerStatefulWidget {
  const MyFormPage({super.key});

  @override
  ConsumerState<MyFormPage> createState() => _MyFormPageState();
}

class _MyFormPageState extends ConsumerState<MyFormPage> {
  @override
  void initState() {
    super.initState();
    // initState 안에서도 ref.read 사용 가능
    ref.read(someNotifierProvider.notifier).init();
  }

  @override
  Widget build(BuildContext context) {
    final data = ref.watch(someNotifierProvider); // 클래스 멤버인 ref 사용
    return Text('$data');
  }
}
```

### 2-3. Consumer (부분 리빌드 최적화용 위젯)

큰 위젯 트리 중 **일부만** 상태 변화에 반응해서 다시 그려지길 원할 때 사용한다. 전체를 `ConsumerWidget`으로 바꾸는 대신 필요한 부분만 `Consumer`로 감싸서 성능을 최적화한다.

```dart
class HeavyPage extends StatelessWidget {
  const HeavyPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const VeryHeavyStaticWidget(), // 상태가 바뀌어도 재빌드 안 됨
        Consumer(
          builder: (context, ref, child) {
            final count = ref.watch(countProvider);
            return Text('$count'); // 오직 이 Text만 재빌드 됨
          },
        ),
      ],
    );
  }
}
```

**한눈에 비교**

| 위젯 | 대체 대상 | 언제 쓰나 |
|------|-----------|-----------|
| `ConsumerWidget` | `StatelessWidget` | 기본 선택 |
| `ConsumerStatefulWidget` | `StatefulWidget` | 컨트롤러, `initState`/`dispose`가 필요할 때 |
| `Consumer` | - | 트리 일부만 리빌드하고 싶을 때 |

## 3. Riverpod의 의존성 주입(DI) 원리

### 3-1. 작동 방식: 선언적 의존성 그래프

객체를 생성하는 책임을 Provider에게 맡기고, 필요한 하위 객체는 `ref.watch` / `ref.read`로 조립한다.

```dart
// 1. 가장 하위 계층: DataSource (API 클라이언트)
final dioProvider = Provider<Dio>((ref) {
  return Dio(BaseOptions(baseUrl: 'https://api.example.com'));
});

// 2. Repository 계층: dioProvider를 주입받음
final authRepositoryProvider = Provider<AuthRepository>((ref) {
  final dio = ref.watch(dioProvider); // 컴파일 타임에 타입 검증됨
  return AuthRepositoryImpl(dio);
});

// 3. UseCase 계층: authRepositoryProvider를 주입받음
final loginUseCaseProvider = Provider<LoginUseCase>((ref) {
  final repository = ref.watch(authRepositoryProvider);
  return LoginUseCase(repository);
});
```

### 3-2. 핵심 장점 3가지

1. **컴파일 타임 안전성**
   - GetX의 `Get.find()`는 `Get.put`으로 미리 등록하지 않고 호출하면 앱 실행 중에 런타임 에러가 난다.
   - Riverpod은 변수(`loginUseCaseProvider`) 자체를 참조하므로, 선언되지 않은 의존성을 호출하는 것 자체가 문법적으로 불가능하다.
2. **지연 초기화 (Lazy Initialization)**
   - Provider는 선언만 해 두면 앱 시작 시 메모리에 올라가지 않는다.
   - UI에서 실제로 `watch` 또는 `read`하는 순간 연쇄적으로 객체가 생성된다.
3. **손쉬운 테스트와 모킹 (Overriding)**
   - 테스트나 개발/운영 서버 전환 시, 상위 `ProviderScope`에서 특정 Provider의 구현체만 갈아끼울 수 있다.

```dart
ProviderScope(
  overrides: [
    // 실제 서버 통신 대신 테스트용 Mock Repository로 즉시 교체
    authRepositoryProvider.overrideWithValue(MockAuthRepository()),
  ],
  child: const MyApp(),
)
```

## 4. 수식어(Modifiers): .autoDispose & .family

Provider 선언 뒤에 붙여서 생명주기와 파라미터 전달을 제어한다.

### 4-1. .autoDispose

해당 Provider를 `watch`하는 위젯이 화면에서 모두 사라지면(예: 뒤로가기) 자동으로 상태를 메모리에서 파기한다. 다시 페이지에 들어가면 초기 상태로 새로 시작한다.

```dart
final detailProvider =
    AsyncNotifierProvider.autoDispose<DetailNotifier, Detail>(DetailNotifier.new);
```

### 4-2. .family

Provider를 호출할 때 외부에서 파라미터를 넘겨줄 때 사용한다. (예: 상세 페이지 조회 시 `userId` 전달)

```dart
// int 타입의 userId를 파라미터로 받는 Provider
final userDetailProvider = FutureProvider.family<User, int>((ref, userId) async {
  return ref.watch(userRepositoryProvider).getUserById(userId);
});

// UI에서 호출 시
final userAsync = ref.watch(userDetailProvider(42));
```

## 5. 실전 예제: 카운터 앱 + 배율(Step) 서비스 DI

Riverpod의 상태 변경(`state = ...`)과 Provider 간 의존성 주입이 어떻게 이어지는지 보여주는 예제

### 전체 구조

1. **Config Provider**: 곱해줄 배율 값을 제공하는 의존성 (DI)
2. **Counter Notifier**: 실제 상태(`state`)를 가지고, 배율을 반영해 숫자를 조작하는 비즈니스 로직
3. **UI (ConsumerWidget)**: 상태를 구독(`ref.watch`)하고, 버튼 클릭 시 메서드를 호출(`ref.read`)

### 5-1. 의존성(Config) Provider 정의

```dart
// 배율을 제공하는 서비스 클래스
class CounterConfigService {
  final int multiplier;
  CounterConfigService({this.multiplier = 1});
}

// 이 서비스를 전역에서 주입받을 수 있도록 Provider 선언
final counterConfigProvider = Provider<CounterConfigService>((ref) {
  // 나중에 서버나 로컬 스토리지에서 가져온 값으로 교체 가능
  return CounterConfigService(multiplier: 2);
});
```

### 5-2. 상태를 관리하는 Notifier 정의

GetX의 Controller에 해당하는 클래스다. `Notifier<int>`를 상속하고, 내부에서 `ref`로 `counterConfigProvider`를 주입받는다.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

class CounterNotifier extends Notifier<int> {
  @override
  int build() {
    // 초기값 설정 (GetX의 onInit과 비슷한 역할)
    return 0;
  }

  void increment() {
    // ref.read로 다른 Provider의 값을 가져와 활용 (DI)
    final config = ref.read(counterConfigProvider);

    // state를 직접 수정하지 않고 새 값으로 통째로 교체 (불변성)
    state = state + config.multiplier;
  }

  void decrement() {
    final config = ref.read(counterConfigProvider);
    state = state - config.multiplier;
  }
}

// Notifier를 UI에 연결하는 NotifierProvider
// .autoDispose: 페이지 이탈 시 메모리 해제
final counterProvider = NotifierProvider.autoDispose<CounterNotifier, int>(() {
  return CounterNotifier();
});
```

### 5-3. UI Layer (ConsumerWidget)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // 1. ref.watch로 상태 구독
    //    숫자가 바뀔 때마다 build가 다시 실행되어 UI가 갱신됨
    final count = ref.watch(counterProvider);
    final config = ref.watch(counterConfigProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Riverpod 카운터 예시')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('현재 배율: ${config.multiplier}배씩 증가'),
            const SizedBox(height: 16),
            Text(
              '$count',
              style: const TextStyle(fontSize: 48, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 24),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton(
                  onPressed: () {
                    // 2. 값을 변경할 때는 ref.read 사용
                    //    .notifier를 붙이면 Notifier 객체에 접근
                    ref.read(counterProvider.notifier).decrement();
                  },
                  child: const Text('감소'),
                ),
                const SizedBox(width: 16),
                ElevatedButton(
                  onPressed: () {
                    ref.read(counterProvider.notifier).increment();
                  },
                  child: const Text('증가'),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

### 5-4. 예제에서 눈여겨볼 포인트

- **GetX와의 차이**
  - GetX: `var count = 0.obs;` 후 `count.value++`로 가변 상태를 직접 수정
  - Riverpod: `state = state + ...`처럼 불변 상태를 새 값으로 교체
- **깔끔한 DI**
  - `CounterNotifier` 안에서 `ref.read(counterConfigProvider)`만 호출하면 외부 서비스를 가져다 쓸 수 있다.
  - 생성자에 객체를 일일이 파라미터로 넘길 필요가 없다.
- **`watch`와 `read`의 명확한 분리**
  - UI를 그릴 때(실시간 구독): `ref.watch(counterProvider)` → 값이 바뀌면 화면 재빌드
  - 버튼을 누를 때(일회성 실행): `ref.read(counterProvider.notifier).increment()` → 재빌드 없이 메서드만 실행

> `build` 메서드 안에서는 `ref.watch`, 이벤트 콜백 안에서는 `ref.read`를 쓰는 것이 기본 규칙이다.
{: .prompt-tip }

## 6. 한 줄 요약

| 개념 | 한 줄 정리 |
|------|-----------|
| `ProviderScope` | 모든 Provider의 상태가 실제로 저장되는 최상위 컨테이너 |
| `Provider` | 객체/상태를 만들어 노출하는 선언 |
| `Ref` | Provider끼리, 위젯과 Provider를 잇는 연결 고리 |
| `Notifier` | `state`와 그 변경 로직을 가진 클래스 |
| `Consumer*` | `ref`를 제공해 상태를 구독하는 위젯 |
| `.autoDispose` | 안 쓰면 자동으로 메모리에서 해제 |
| `.family` | Provider에 외부 파라미터 전달 |