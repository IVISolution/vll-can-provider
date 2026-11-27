# VLL Features

> Why VLL was made and how it is designed is in [concept.md](concept.md). This document covers only how to write rules. Korean: [ko/features.md](ko/features.md)

## 1. Features at a glance

| # | Feature | Syntax | Requirement (concept document, chapter 2) | Section |
|---|---|---|---|---|
| F1 | CAN signal input | `<Message>.<Signal>: nickname` in `inputs` | R1 | 3 |
| F2 | VSS signal input | `<VSS path>: nickname` in `inputs` | R2 | 3 |
| F3 | Multiple input signals | One signal per line in `inputs`, several signals together | R1 | 2 |
| F4 | Operators | Arithmetic, bitwise, comparison, logical, ternary | R1 | 5.3 |
| F5 | Assignment operators | `= += -= *= /= ...` (accumulate with `this += x`) | — | 5.1 |
| F6 | Decision table (`match`) | `match (a, b) { (1, _) => ..., (_, _) => ... }` | R3 | 5.4 |
| F7 | Guards (`when`) | `when (condition) => ...`, `else => ...` inside `match` | R3 | 5.5 |
| F8 | Change detection | `"ON" @is_updated`, `"ON" @changed_from("OFF")` | — | 5.6 |
| F9 | Update interval | `interval(nickname)` | — | 5.7 |
| F10 | CAN output | `actuate.outputs` + `logic` | R4 | 4 |
| F11 | Arrays | `[1, 2]`, `arr[0]` | — | 5.2 |
| F12 | Time conditions | `@after`, `@before`, `@at`, `@repeat` | — | 5.8 |

## 2. Where rules go

Rules are written under a signal in a VSS overlay file (`.vspec`) with the `vll` key. The standard VSS files are not changed.

```yaml
Vehicle.Speed:
  vll:
    derive:                         # calculate this VSS value from other signals
      inputs:                       # inputs: <signal address>: <nickname>
        - WHL_SPD11.WHL_SPD_FL: fl
        - WHL_SPD11.WHL_SPD_FR: fr
        - WHL_SPD11.WHL_SPD_RL: rl
        - WHL_SPD11.WHL_SPD_RR: rr
      logic: |                      # logic: VLL expressions
        this = (fl + fr + rl + rr) / 4
```

| Key | Meaning |
|---|---|
| `vll` | Start of the VLL rule. A vss-tools extended attribute, merged with standard VSS by `vspec export ... -e vll` |
| `derive` | Calculates the value of this VSS signal from input signals |
| `actuate` | When this VSS signal is updated, calculates CAN signals and sends them (section 4) |
| `inputs` / `outputs` | Input signals to watch (the ingredients) / output CAN signals. One `<signal address>: <nickname>` per line; several lines for several signals |
| `logic` | VLL expressions that compute the value (the steps) |

An overlay can redefine the `datatype` of a VSS signal. For example, if fractional values are needed, the standard VSS `uint32` can become `double`.

## 3. Signal addresses and nicknames

| Address | Format | Notes |
|---|---|---|
| CAN signal | `<Message>.<Signal>` | The message name and the signal name in the DBC file, joined by a dot. Example: `WHL_SPD11.WHL_SPD_FL` |
| VSS signal | A VSS path such as `Vehicle.Speed` | Only in `inputs`. That VSS signal must also have a `vll` rule. The input VSS signal is calculated first, then this rule. Do not create circular references where signals use each other as input |

```text
BO_ 902 WHL_SPD11: 8 ABS                                                  ← message name: WHL_SPD11
 SG_ WHL_SPD_FL : 0|14@1+ (0.03125,0.0) [0.0|511.96875] "km/h" ...         ← signal name: WHL_SPD_FL
```

Names defined in the DBC are used as they are, so addresses are not defined separately. The build checks them against the DBC and reports an error for an unknown name.

- `outputs` take only CAN signals.
- Nicknames have the form `[a-zA-Z_][a-zA-Z0-9_]*`. Reserved words (`this`, `match`, `when`, `else`, `true`, `false`, `_`, `interval`, the time units `ms`, `s`, `sec`, `m`, `min`, `h`, `hr`, `hour`, and C++ keywords) and duplicates within one signal are not allowed. This is checked at build time.
- `this` is the VSS signal the rule belongs to. In `derive`, the value assigned to `this` becomes the new value of the signal, and `this` in an expression is the previous value of the signal. In `actuate`, `this` is the updated value of the signal (section 4).

## 4. derive and actuate

| Group | Direction | Keys | Used for |
|---|---|---|---|
| `derive` | CAN and VSS signals → this VSS signal | `inputs` + `logic` | Sensors, and the current value of actuators |
| `actuate` | This VSS signal → CAN signals | `outputs` + `logic` | Actuators |

To use the value of this VSS signal in another VSS signal, list this signal in the `derive` `inputs` of that signal, not in `actuate` (section 3).

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

- `actuate` runs its `logic` when this VSS signal is updated, and when the value assigned to an output nickname changes, sends it to that CAN signal in `outputs`. `this` in an expression is the updated value (for an actuator, the target value requested by the app). If the logic produces no value, nothing is sent.
- Apps request a target value (Actuate) through the single actuator path of standard VSS. No separate signal is written for the target value.

## 5. Logic syntax

### 5.1 Statements

Logic is a sequence of assignments. The `;` between statements can be omitted; using `;` is recommended when several statements are on one line. Text after `#` or `//` is a comment.

```text
this = a + b
this += 1
```

Assignment operators: `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`

### 5.2 Values and types

| Value | Example |
|---|---|
| Integer | `10`, `0x1F` |
| Floating point | `0.5` (exponent notation such as `1e-5` is not supported) |
| Boolean | `true`, `false` |
| String | `"ON"` |
| Array | `[1, 2, 3]`, accessed as `arr[0]` |

- The types are uint64, int64, double, bool, string and arrays of them. Structs (VSS struct) are not supported.
- VSS integer types (uint8 to uint64, int8 to int64) are handled as uint64 or int64 by sign, and float as double.
- Numeric types (uint64, int64, double) and bool can be mixed in calculations, and the result is converted to the type of the assigned signal. For example, assigning a double value to an int64 signal drops the fraction. This conversion is not reported as an error, and the range of the signal type (for example 300 for uint8) is not checked. Write unit conversions in the expression (for example `this * 3.6`).
- string cannot be mixed with other types. Mixing them shows up at build time as a compile error in the generated C++ code.
- For arrays, only assigning an array value and indexing a nickname (`arr[0]`) are supported. Operations between arrays are not supported.

### 5.3 Operators

| Kind | Operators |
|---|---|
| Unary | `-`, `!`, `++`, `--` |
| Arithmetic | `*`, `/`, `%`, `+`, `-` |
| Shift | `<<`, `>>` |
| Comparison | `<`, `<=`, `>`, `>=`, `==`, `!=` |
| Bitwise | `&`, `\|`, `^` |
| Logical | `&&`, `\|\|` |
| Ternary | `condition ? value : value` |

Precedence is the same as in C/C++, because expressions are copied into C++ code as they are. When mixing comparison, bitwise and logical operators, use parentheses to make the order explicit. Example: `(a & 1) == 1`, `a || (b && c)`.

The available operators differ by type (○ allowed, ✕ not allowed).

| Operator | uint64 | int64 | double | bool | string |
|---|---|---|---|---|---|
| Increment, decrement `++` `--` | ○ | ○ | ○ | ✕ | ✕ |
| Arithmetic `+` `-` `*` `/` | ○ | ○ | ○ | ○ | `+` only |
| Remainder `%` | ○ | ○ | ✕ | ✕ | ✕ |
| Bitwise `&` `\|` `^` `<<` `>>` | ○ | ○ | ✕ | ○ | ✕ |
| Comparison `<` `>` `<=` `>=` `==` `!=` | ○ | ○ | ○ | ○ | ○ (the result cannot be assigned to a string) |
| Logical `&&` `\|\|` | ○ | ○ | ○ | ○ | ✕ |
| Ternary `? :` | ○ | ○ | ○ | ○ | ○ |
| Assignment | all | all | `=` `+=` `-=` `*=` `/=` | `=` `&=` `\|=` `^=` | `=` `+=` |

- Logical operators on numbers treat any non-zero value as true. In arithmetic, bool counts `true` as 1 and `false` as 0.
- Both results of the ternary operator must be numbers, or both must be strings.
- When the divisor of `/` or `%` is 0, no value is produced. The signal keeps its previous value, and in `actuate` nothing is sent to CAN.

### 5.4 Decision table (`match`)

`match` writes a decision table, which lists the result for each combination of input values, as it is. Each line is a table row: "if the values are these, the result is that." The input values in parentheses are compared with the pattern of each line, and the value to the right of `=>` on the matching line is the result. The form is the same as Rust's `match`.

```text
this = match (manual, prndl) {
    (0, 1) => 126,
    (1, _) => 14 - prndl,
    (_, _) => 0
}
```

- `match` is used only as the whole right-hand side of an assignment. It cannot be combined with other operators.
- The n-th position of a pattern is compared with the n-th input in the `match` parentheses.
- A pattern position holds a value (number, string, boolean, another nickname) or the wildcard `_`. A `!` before a value means "not that value". Negative numbers (`-1`) cannot be used in patterns; compare them with `when`.
- A result position can hold an expression, a `{ }` block or a `when` block (5.5).
- If no line matches, no value is produced. To handle every input, put a line with `_` in every position last.
- Write the patterns of lines with different results so that they do not overlap. With overlapping patterns, the result may not follow the declaration order. Example: with `(0, 1) => 1`, `(1, _) => 2`, `(1, 4) => 1` in this order, the input `(1, 4)` would give 2 in declaration order, but the actual result is 1.

### 5.5 Guards (`when`, `else`)

A `when` block in the result position of `match` splits again by condition after the pattern has matched. This is a match guard: Rust writes `if`, VLL writes `when`.

```text
this = match (active) {
  (1) => {
    when (force < -0.00001) => true,
    else => false
  },
  (_) => false
}
```

`when` branches are checked from the top, and if none matches, `else` is used. If there is no `else` and no `when` matches, no value is produced, and in `actuate` nothing is sent to CAN (the temperature setting example in section 3.3 of [concept.md](concept.md)).

### 5.6 Change detection (`@is_updated`, `@changed_from`)

Placed after a pattern value, these make the pattern match only when the input has just become that value. Use them to change the result when one particular input among several changes.

| Syntax | Matches when |
|---|---|
| `"ON" @is_updated` | The input has just changed to `"ON"` |
| `"ON" @changed_from("OFF")` | The input has just changed from `"OFF"` to `"ON"` |
| `"ON" @changed_from(_)` | The input has just changed from any other value to `"ON"`. Same as `@is_updated` |

```text
this = match (ign, state) {
  ("ON" @changed_from("OFF"), "TURNING_OFF") => "TURNING_ON",
  (_, _) => state
}
```

### 5.7 Update interval (`interval`)

`interval(nickname)` is the time since the last update of that input: the time (ms) between its current and previous update.

- It does not measure time periodically; it is calculated when the input value changes. The same value arriving again is not an update, so the whole time until the value changes is reflected at once.
- The first update has no previous update, so no value is produced.

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

To accumulate with `this +=`, give a start value with `default`. Without a start value, `this` has no value and no value is produced.

This example is an approximation. It multiplies the time since the previous update by the newly arrived speed, not by the speed during that time. When the same speed lasts a long time and then changes, the error is large. Example: after driving at 100 km/h for an hour and then stopping, the distance of that hour is not added.

### 5.8 Time conditions (`@after`, `@before`, `@at`, `@repeat`)

These match depending on how long a value has been held, or repeat a result periodically.

| Syntax | Position | Meaning |
|---|---|---|
| `"ON" @after(200ms)` | After a pattern value | Matches when the value has been held for 200 ms or more |
| `"ON" @before(200ms)` | After a pattern value | Matches within 200 ms after changing to the value |
| `"ON" @at(200ms)` | After a pattern value | Matches at the moment 200 ms after changing to the value |
| `=> this + 1 @repeat(1s)` | After a result | Repeats the result every second |

The time units are `ms`, `s`/`sec`, `m`/`min`, `h`/`hr`/`hour`. Time unit names are reserved words and cannot be used as nicknames.

## 6. Examples

### 6.1 Gear lever (R3, decision table)

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

As this is an example, some automatic-mode values (such as L) are left out.

### 6.2 Emergency braking (R3, state + value comparison)

Split by state, then by value.

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
