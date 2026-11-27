# VLL 기능 설명

> VLL을 왜 만들었고 어떻게 설계했는지는 [concept.md](concept.md)에 있다. 이 문서는 규칙을 어떻게 작성하는지만 다룬다. English: [../features.md](../features.md)

## 1. 기능 한눈에 보기

| # | 기능 | 문법 | 요구사항(컨셉 문서 2장) | 절 |
|---|---|---|---|---|
| F1 | CAN 신호 입력 | `inputs`에 `<Message>.<Signal>: 별명` | R1 | 3 |
| F2 | VSS 신호 입력 | `inputs`에 `<VSS 경로>: 별명` | R2 | 3 |
| F3 | 여러 입력 신호 | `inputs`에 한 줄에 하나씩, 여러 신호를 함께 입력 | R1 | 2 |
| F4 | 연산 | 산술, 비트, 비교, 논리, 삼항 | R1 | 5.3 |
| F5 | 대입 연산 | `= += -= *= /= ...` (`this += x`로 누적) | — | 5.1 |
| F6 | 결정 테이블 (`match`) | `match (a, b) { (1, _) => ..., (_, _) => ... }` | R3 | 5.4 |
| F7 | 조건 분기 | `match` 안의 `when (조건) => ...`, `else => ...` | R3 | 5.5 |
| F8 | 값 변화 감지 | `"ON" @is_updated`, `"ON" @changed_from("OFF")` | — | 5.6 |
| F9 | 갱신 간격 | `interval(별명)` | — | 5.7 |
| F10 | CAN 출력 | `actuate.outputs` + `logic` | R4 | 4 |
| F11 | 배열 | `[1, 2]`, `arr[0]` | — | 5.2 |
| F12 | 시간 조건 | `@after`, `@before`, `@at`, `@repeat` | — | 5.8 |

## 2. 규칙을 작성하는 곳

규칙은 VSS overlay 파일(`.vspec`)의 신호 아래에 `vll` 키로 작성한다. 표준 VSS 파일은 수정하지 않는다.

```yaml
Vehicle.Speed:
  vll:
    derive:                         # 다른 신호로부터 이 VSS 값을 계산한다
      inputs:                       # 입력: <신호 주소>: <별명>
        - WHL_SPD11.WHL_SPD_FL: fl
        - WHL_SPD11.WHL_SPD_FR: fr
        - WHL_SPD11.WHL_SPD_RL: rl
        - WHL_SPD11.WHL_SPD_RR: rr
      logic: |                      # 로직: VLL 식
        this = (fl + fr + rl + rr) / 4
```

| 키 | 뜻 |
|---|---|
| `vll` | VLL 규칙의 시작. vss-tools 확장 속성이며 `vspec export ... -e vll`로 표준 VSS와 병합한다 |
| `derive` | 입력 신호로부터 이 VSS 신호의 값을 계산한다 |
| `actuate` | 이 VSS 신호가 갱신되면 CAN 신호를 계산해 송신한다(4절) |
| `inputs` / `outputs` | 입력 신호 / 출력 CAN 신호 목록. 한 줄에 `<신호 주소>: <별명>` 하나씩, 여러 줄로 여러 신호를 적는다 |
| `logic` | 값을 계산하는 VLL 식 |

overlay에서 VSS 신호의 `datatype`을 재정의할 수 있다. 예를 들어 소수 값을 다뤄야 하면 표준 VSS의 `uint32`를 `double`로 바꿀 수 있다.

## 3. 신호 주소와 별명

| 주소 | 형식 | 비고 |
|---|---|---|
| CAN 신호 | `<Message>.<Signal>` | DBC 파일의 메시지 이름과 신호 이름을 점으로 잇는다. 예: `WHL_SPD11.WHL_SPD_FL` |
| VSS 신호 | `Vehicle.Speed`처럼 VSS 경로 | `inputs`에만 쓴다. 그 VSS 신호에도 `vll` 규칙이 있어야 한다. 입력 VSS 신호가 먼저 계산된 뒤 이 규칙이 계산된다. 서로를 입력으로 쓰는 순환 참조는 만들지 않는다 |

```text
BO_ 902 WHL_SPD11: 8 ABS                                                  ← 메시지 이름: WHL_SPD11
 SG_ WHL_SPD_FL : 0|14@1+ (0.03125,0.0) [0.0|511.96875] "km/h" ...         ← 신호 이름: WHL_SPD_FL
```

DBC에 정의된 이름을 그대로 쓰므로 주소를 따로 정의하지 않는다. 빌드할 때 DBC와 대조해, 없는 이름이면 오류가 난다.

- `outputs`에는 CAN 신호만 적는다.
- 별명은 `[a-zA-Z_][a-zA-Z0-9_]*` 형식이다. 예약어(`this`, `match`, `when`, `else`, `true`, `false`, `_`, `interval`, 시간 단위 `ms`·`s`·`sec`·`m`·`min`·`h`·`hr`·`hour`, C++ 키워드)와 한 신호 안의 중복은 쓸 수 없다. 빌드 단계에서 검사한다.
- `this`는 규칙이 붙은 VSS 신호다. `derive`에서는 `this`에 대입한 값이 그 신호의 새 값이 되고, 식 안의 `this`는 그 신호의 직전 값이다. `actuate`에서 `this`는 그 신호의 갱신된 값이다(4절).

## 4. derive와 actuate

| 그룹 | 방향 | 키 | 쓰는 곳 |
|---|---|---|---|
| `derive` | CAN·VSS 신호 → 이 VSS 신호 | `inputs` + `logic` | sensor, actuator의 현재값 |
| `actuate` | 이 VSS 신호 → CAN 신호 | `outputs` + `logic` | actuator |

이 VSS 신호의 값을 다른 VSS 신호에 쓰려면 `actuate`가 아니라 그 신호의 `derive` `inputs`에 이 신호를 적는다(3절).

```yaml
Vehicle.Cabin.HVAC.IsAirConditioningActive:   # actuator
  vll:
    derive:
      inputs:
        - FATC11.CF_Fatc_AcnRqSwi: sw
      logic: |
        this = sw
    actuate:
      outputs:
        - TMU11.CF_Tmu_AirconCtr: out1
      logic: |
        out1 = this
```

- `actuate`는 이 VSS 신호가 갱신되면 `logic`을 실행하고, 출력 별명에 대입한 값이 바뀌면 `outputs`의 그 CAN 신호로 송신한다. 식 안의 `this`는 갱신된 값이다(actuator면 앱이 요청한 목표값). 로직이 값을 만들지 않으면 송신하지 않는다.
- 앱은 표준 VSS의 actuator 경로 하나로 목표값을 요청(Actuate)한다. 목표값을 위한 신호는 따로 작성하지 않는다.

## 5. 로직 문법

### 5.1 문장

로직은 대입문의 나열이다. 문장 사이의 `;`는 생략할 수 있으며, 한 줄에 여러 문장을 쓸 때는 `;`로 구분하는 것을 권장한다. `#`나 `//` 뒤는 주석이다.

```text
this = a + b
this += 1
```

대입 연산자: `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`

### 5.2 값과 타입

| 값 | 예 |
|---|---|
| 정수 | `10`, `0x1F` |
| 실수 | `0.5` (지수 표기 `1e-5`는 쓸 수 없다) |
| 불리언 | `true`, `false` |
| 문자열 | `"ON"` |
| 배열 | `[1, 2, 3]`, 접근은 `arr[0]` |

- 타입은 uint64, int64, double, bool, string과 그 배열이다. 구조체(VSS struct)는 지원하지 않는다.
- VSS의 정수 타입(uint8–uint64, int8–int64)은 부호에 따라 uint64나 int64로, float는 double로 다룬다.
- 숫자 타입(uint64, int64, double)과 bool은 서로 섞어 계산할 수 있고, 결과는 대입하는 신호의 타입으로 변환된다. 예: double 값을 int64 신호에 대입하면 소수점 이하는 버려진다. 이 변환은 오류로 알리지 않고, 신호 타입의 범위(예: uint8에 300)도 검사하지 않는다. 단위 환산은 식에 적는다(예: `this * 3.6`).
- string은 다른 타입과 섞을 수 없다. 섞으면 빌드 단계에서 생성된 C++ 코드의 컴파일 오류로 나온다.
- 배열은 배열 값 대입과 별명의 인덱스 접근(`arr[0]`)만 지원한다. 배열끼리의 연산은 지원하지 않는다.

### 5.3 연산자

| 종류 | 연산자 |
|---|---|
| 단항 | `-`, `!`, `++`, `--` |
| 산술 | `*`, `/`, `%`, `+`, `-` |
| 시프트 | `<<`, `>>` |
| 비교 | `<`, `<=`, `>`, `>=`, `==`, `!=` |
| 비트 | `&`, `\|`, `^` |
| 논리 | `&&`, `\|\|` |
| 삼항 | `조건 ? 값 : 값` |

우선순위는 C/C++과 같다. 식이 C++ 코드로 그대로 옮겨지기 때문이다. 비교·비트·논리 연산자를 섞어 쓸 때는 괄호로 계산 순서를 명시한다. 예: `(a & 1) == 1`, `a || (b && c)`.

타입마다 쓸 수 있는 연산자가 다르다(○ 가능, ✕ 불가).

| 연산자 | uint64 | int64 | double | bool | string |
|---|---|---|---|---|---|
| 증감 `++` `--` | ○ | ○ | ○ | ✕ | ✕ |
| 산술 `+` `-` `*` `/` | ○ | ○ | ○ | ○ | `+`만 |
| 나머지 `%` | ○ | ○ | ✕ | ✕ | ✕ |
| 비트 `&` `\|` `^` `<<` `>>` | ○ | ○ | ✕ | ○ | ✕ |
| 비교 `<` `>` `<=` `>=` `==` `!=` | ○ | ○ | ○ | ○ | ○ (결과를 string에 대입할 수는 없다) |
| 논리 `&&` `\|\|` | ○ | ○ | ○ | ○ | ✕ |
| 삼항 `? :` | ○ | ○ | ○ | ○ | ○ |
| 대입 | 전부 | 전부 | `=` `+=` `-=` `*=` `/=` | `=` `&=` `\|=` `^=` | `=` `+=` |

- 숫자에 논리 연산자를 쓰면 0이 아닌 값을 참으로 본다. bool을 숫자로 계산하면 `true`는 1, `false`는 0이다.
- 삼항 연산자의 두 결과는 둘 다 숫자이거나 둘 다 string이어야 한다.
- `/`, `%`에서 나누는 값이 0이면 값을 만들지 않는다. 신호는 직전 값을 유지하고, `actuate`면 CAN으로 송신하지 않는다.

### 5.4 결정 테이블 (`match`)

입력 값의 조합별 결과를 적은 결정 테이블(decision table)을 그대로 옮긴다. 한 줄이 표의 한 행이며, 괄호 안의 입력 값들을 각 줄의 패턴과 비교해 일치하는 줄의 `=>` 오른쪽 값을 결과로 쓴다. Rust의 `match`와 같은 형태다.

```text
this = match (manual, prndl) {
    (0, 1) => 126,
    (1, _) => 14 - prndl,
    (_, _) => 0
}
```

- `match`는 대입의 오른쪽 전체로만 쓴다. 다른 연산자와 섞을 수 없다.
- 패턴의 n번째 자리는 `match` 괄호 안의 n번째 입력과 비교한다.
- 패턴 자리에는 값(숫자, 문자열, 불리언, 다른 별명)이나 와일드카드 `_`를 쓴다. 값 앞에 `!`를 붙이면 "그 값이 아닐 때"다. 음수(`-1`)는 패턴에 쓸 수 없으므로 `when`으로 비교한다.
- 결과 자리에는 식, `{ }` 블록, `when` 블록(5.5절)을 쓸 수 있다.
- 일치하는 줄이 없으면 값을 만들지 않는다. 모든 입력을 처리하려면 마지막에 모든 자리가 `_`인 줄을 둔다.
- 서로 다른 결과를 내는 줄의 패턴은 겹치지 않게 작성한다. 패턴이 겹치면 선언 순서와 다른 결과가 나올 수 있다. 예: `(0, 1) => 1`, `(1, _) => 2`, `(1, 4) => 1` 순서로 쓰면, 입력 `(1, 4)`는 선언 순서대로라면 2이지만 실제 결과는 1이다.

### 5.5 조건 분기 (`when`, `else`)

`match`의 결과 자리에 `when` 블록을 두면, 패턴이 일치한 뒤 조건식으로 다시 나눈다. Rust의 match guard(`if`)에 해당한다.

```text
this = match (active) {
  (1) => {
    when (force < -0.00001) => true,
    else => false
  },
  (_) => false
}
```

`when`은 위에서부터 차례로 검사하고, 일치하는 것이 없으면 `else`를 쓴다. `else`가 없고 일치하는 `when`도 없으면 값을 만들지 않는다. `actuate`에서는 CAN으로 송신하지 않는다([concept.md](concept.md) 3.3절의 온도 설정 예).

### 5.6 값 변화 감지 (`@is_updated`, `@changed_from`)

패턴 값 뒤에 붙여, 입력이 "방금" 그 값이 된 경우에만 일치하게 한다. 여러 입력 중 특정 입력의 변화를 계기로 결과를 바꿀 때 쓴다.

| 문법 | 일치하는 경우 |
|---|---|
| `"ON" @is_updated` | 입력이 방금 `"ON"`으로 바뀌었을 때 |
| `"ON" @changed_from("OFF")` | 입력이 방금 `"OFF"`에서 `"ON"`으로 바뀌었을 때 |
| `"ON" @changed_from(_)` | 입력이 방금 다른 어떤 값에서 `"ON"`으로 바뀌었을 때. `@is_updated`와 같다 |

```text
this = match (ign, state) {
  ("ON" @changed_from("OFF"), "TURNING_OFF") => "TURNING_ON",
  (_, _) => state
}
```

### 5.7 갱신 간격 (`interval`)

`interval(별명)`은 그 입력의 현재 갱신과 직전 갱신 사이의 시간(ms)이다.

- 주기적으로 시간을 재는 것이 아니라, 입력값이 바뀔 때 계산한다. 같은 값이 계속 들어오면 갱신이 아니므로, 값이 바뀔 때 그동안의 시간이 한 번에 반영된다.
- 첫 갱신에는 직전 갱신이 없으므로 값을 만들지 않는다.

```yaml
Vehicle.TraveledDistanceSinceStart:
  datatype: double
  default: 0
  vll:
    derive:
      inputs:
        - Vehicle.Speed: speed
      logic: |
        this += speed * interval(speed) / 3600   # km/h × ms / 3600 = m
```

`this +=`로 누적하려면 `default`로 시작값을 준다. 시작값이 없으면 `this`에 값이 없어 값을 만들지 않는다.

이 예제는 근사값이다. 직전 갱신부터의 시간에 그 사이의 속도가 아니라 새로 들어온 속도를 곱한다. 같은 속도가 오래 이어진 뒤 바뀌면 오차가 크다. 예: 100 km/h로 1시간 달린 뒤 0이 되면 그 1시간의 거리가 더해지지 않는다.

### 5.8 시간 조건 (`@after`, `@before`, `@at`, `@repeat`)

값이 얼마나 유지됐는지에 따라 일치시키거나, 결과를 주기적으로 반복한다.

| 문법 | 위치 | 뜻 |
|---|---|---|
| `"ON" @after(200ms)` | 패턴 값 뒤 | 그 값이 200ms 이상 유지됐을 때 일치한다 |
| `"ON" @before(200ms)` | 패턴 값 뒤 | 그 값으로 바뀐 뒤 200ms 이내일 때 일치한다 |
| `"ON" @at(200ms)` | 패턴 값 뒤 | 그 값으로 바뀐 뒤 200ms가 되는 시점에 일치한다 |
| `=> this + 1 @repeat(1s)` | 결과 뒤 | 결과를 1초 주기로 반복한다 |

시간 단위는 `ms`, `s`/`sec`, `m`/`min`, `h`/`hr`/`hour`다. 시간 단위 이름은 예약어라 별명으로 쓸 수 없다.

## 6. 예제

### 6.1 변속 레버 (R3, 결정 테이블)

```yaml
Vehicle.Powertrain.Transmission.SelectedGear:
  vll:
    derive:
      inputs:
        - ECMPRDNL2.ManualMode: manual
        - ECMPRDNL2.PRNDL2: prndl
      logic: |
        this = match (manual, prndl) {
          (0, 1) => 126,
          (0, 2) => -1,
          (0, 3) => 0,
          (0, 4) => 127,
          (1, _) => 14 - prndl,
          (_, _) => 0
        }
```

설명을 위한 예시라 자동 모드의 일부 값(L 등)은 적지 않았다.

### 6.2 긴급 제동 (R3, 상태 + 값 비교)

상태로 먼저 나누고, 그 안에서 값의 크기로 다시 나눈다.

```yaml
Vehicle.ADAS.EBA.IsEngaged:
  vll:
    derive:
      inputs:
        - PRE_COLLISION.PRECOLLISION_ACTIVE: active
        - PRE_COLLISION.FORCE: force
      logic: |
        this = match (active) {
          (1) => {
            when (force < -0.00001) => true,
            else => false
          },
          (_) => false
        }
```
