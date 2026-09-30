## 1. Shell and layout

- One `<textarea>` fills 100vw × 100vh with no border, outline, title, or toolbar. Only a thin **bottom bar** exists.
- The bar holds suggestion "chips" and is empty when there are none.
- Inline ghost text is drawn by a **mirror `<div>`** layered under or over a transparent-background textarea. It copies the textarea's font, padding, wrapping, and scroll position. The mirror contains the text before the caret, then a grey `<span>` with the ghost text.
- Insertion uses `document.execCommand('insertText')` so native undo/redo keeps working. `setRangeText` is the fallback.
- The theme follows `prefers-color-scheme`. The font is monospace.
- Content autosaves to `localStorage`.
- Ghost rendering technique: textarea + mirror div. Native editing and undo, and robust wrapping.
- Anything else in the bar: Chips plus a faint key hint (`Tab to accept`) when suggestions exist.
- Tab when there's no suggestion: Insert a real tab

---

## 2. Architecture (all in one `<script>`)

1. **Expression finder** isolates the math expression ending at the caret.
2. **Tokenizer** handles numbers, operators, identifiers, unit tokens, and implicit multiplication.
3. **Parser** is a precedence-climbing parser that produces an AST.
4. **Evaluator** works on `Quantity {value, dims[], unit info}`. Plain numbers are dimensionless quantities. It runs twice, in radians mode and degrees mode, only if a bare-number trig argument exists.
5. **Unit engine** holds the unit table, prefix rules, dimension arithmetic, and derived-unit naming.
6. **Formatter** handles the shortest-within-tolerance rounding and the longer-decimal ladder.
7. **Suggestion engine** builds an ordered list of `{insertText, label, inlineEligible}`.
8. **UI** covers the ghost, the chips, key handling, and click handling.

---

## 3. Finding the expression and deciding when to fire

The ghost appears only when the caret is at the end of a line (or followed only by whitespace). Two shapes:

- **No `=` typed:** the suggestion inserts `=result` (e.g. `1+1` → ghost `=2`).
- **`=` typed:** the suggestion inserts only the remainder (e.g. `1*5+9=` → ghost `14`).
- If the user typed a partial result (e.g. `=0.01`), suggestions are matched against it as a prefix and only the remainder is inserted.

No suggestion fires for a lone literal like `5`. Constants (`pi`), functions, operators, and unit expressions do fire.

How to locate the start of the expression in prose like "Total is 1+1": Longest valid suffix parse. Scan left from the caret and keep the longest token run that parses, so prose words are skipped automatically.
Spacing of inserted text: Mirror the user's style. `1 + 1` → ` = 2`, `1+1` → `=2`. Units follow the user's number/unit spacing (`5m` vs `5 m`).
Typed result doesn't match any suggestion** (e.g. `1/97=0.02`): Show a chip that replaces the typed value with the correct one.
Should a bare unit quantity (`5km`) with no operator show conversions? Yes, the bar shows conversions (`=3.11mi`, `=5000m`, …) but the inline ghost does not show anything.

---

## 4. Math grammar and precedence

Supported: `+ - * / % ^ ** !`, parentheses, `log`, `ln`, `log2`, `log_2`, `log5(3)`, `sqrt`, `cbrt`, `sin cos tan csc sec cot`, inverses (`arcsin`, `asin`, `sin-1`, `sin^-1`, `sin⁻¹`), `nPr`, `nCr`, and the constants `e`, `pi`/`π`. `^` and `**` are the same right-associative power operator. The default precedence is, from tightest to loosest: `!`, then `^`/`**`, then unary minus, then implicit multiplication, then `* / %`, then `+ -`.

Function names autocomplete as you type (`sq` → ghost `rt(`).

- What does bare `log` mean?: `log` = base 10, `ln` = natural, `logN` / `log_N` = base N.

What does `%` mean?: Context-dependent. Infix with a right operand (`10%3`) is modulo, and postfix (`50%`, `50%)`, `50%+1`) is percent (÷100).

- Function application without parentheses (`sin 2pi`, `sin pi`): The argument is the following implicit-multiplication run. `sin 2pi` = sin(2π), but `sin 2 * 3` = (sin 2)·3.

Implicit multiplication versus division (`1/2pi`): Implicit multiplication binds tighter, so 1/(2π). This matches how people write it.

Unary minus and power (`-2^2`): −4 (standard math). `2^-1` is also allowed.

Is `sin-1` the inverse or `sin` of −1?: `sin-1` is the inverse only when directly followed by `(` or a number with no space (`sin-1(0.5)`, `sin-1 0.5`). `sin -1` (with a space) and `sin(-1)` mean sine of negative one.

Factorial domain(`!`): Non-negative integers up to 170. Non-integers use the gamma function. Anything else gives no suggestion.

nPr / nCr syntax: `nPr(5,2)`, `nCr(5,2)`, `5 nPr 2`, `5 nCr 2`, plus `P(5,2)` / `C(5,2)`

Q16 – Constants: `e`, `pi`, `π`. `1e5` is scientific notation, and `2e` is 2·e.

Q17 – Invalid or complex results (`sqrt(-1)`, `log(0)`, `asin(2)`, `0/0`, `tan(pi/2)`): Support complex numbers (`i`).

---

## 5. Radians and degrees

- If any trig function has a bare-number argument, evaluate in both modes. Radians is the first chip and degrees is the second.
- Degree mode applies to the whole expression, as in your `tan(sin(…))**(…)` example.
- The same applies to inverse trig outputs: `asin(0.5)` gives 0.5236 (rad), then 30 (deg).
- Explicit angle units (`sin(30deg)`, `sin(30°)`, `sin(pi rad)`) produce a single result.
- Duplicates are removed: `sin 0` shows one chip.
- Example: `sin pi` gives `=0` (radians, since 1.2e-16 snaps to 0), then `=0.055` (degrees).
- Example: `sin 90` gives `=0.9`, then `=1`.

Chip labeling for the two modes: Unlabeled, since the order is the signal. A faint `rad` / `deg` tag appears in each chip.

Mixed explicit and bare angles (`sin(30°) + cos(1)`): The bare `1` is still ambiguous, so two chips appear (rad and deg for the bare argument only).

---

## 6. Number formatting and the decimal ladder

- **Arithmetic** uses Decimal.js arbitrary precision.
- **Exactness test:** if the cleaned value is a terminating decimal, it is exact. It gets no "longer" chips (`1+1` → `=2` only).
- **Inline ghost:** the shortest decimal rounding whose relative error is ≤ **5%**. Integers are never rounded. This fits all your examples:
  - 1/97: 0.0103 → `0.01` (3%).
  - sin 90°: 0.894 → `0.9` (0.7%).
  - sin π°: 0.0548 → `0.055`, because `0.05` is 8.7% off.
- **Longer ladder in the bar (never inline):** more precise versions at 3, 5, and 8 significant figures, deduplicated. For 1/97 these are `0.0103`, `0.010309`, `0.010309278`.
- If the user typed `=0.01`, the bar chips show only the extra digits highlighted (`03`, …) and insert the remainder.
- Very large or small values use scientific notation (≥1e15 or <1e-6).

Rounding rule for the inline value: Shortest decimal within 5% relative error.

Should inline approximations be marked? No, plain `=0.01`

Precision and arithmetic engine: Decimal.js arbitrary precision.

---

## 7. Units engine

### Design

- Each unit is stored as `{names[], symbols[], dims, factorToSI, offset?, prefixable: 'all' | 'large' | 'none'}`.
- Dimension vector: length, mass, time, current, temperature, data, angle.
- Symbols are **case-sensitive** (`m` ≠ `M`, `Mb` ≠ `MB`, `mB` ≠ `MB`). Full names are case-insensitive, with plurals and alternate spellings.
- Prefix parsing is greedy, but an **exact base-unit match wins** over prefix+unit.
- Unit names and symbols also autocomplete as you type (`5 kilo` → `gram`/`meter`/…).

### Coverage

- **Mass:** gram + all prefixes, ounce, pound.
- **Area:** square foot/meter/kilometer/mile/yard/inch, acre. Also derived from `ft^2`, `ft²`, `sq ft`.
- **Data:** bit and byte + prefixes, optionally per second (`Mbps`, `MB/s`).
- **Energy:** joule + prefixes, calorie, watt-hour + prefixes.
- **Frequency:** hertz + prefixes.
- **Fuel economy:** mpg, km/L.
- **Length:** mile, yard, foot, inch, meter + prefixes.
- **Angle:** degree, radian, gradian.
- **Pressure:** pascal + prefixes, bar + prefixes, atm, PSI.
- **Speed:** mph, ft/s, m/s, kph, knot.
- **Temperature:** °F, °C, K.
- **Time:** second + prefixes, minute, hour, day, week, year, decade, century.
- **Volume:** gallon, pint, quart, cup, tbsp, tsp, m³, liter + prefixes, ft³, in³, fl oz.
- **Power:** watt + prefixes, HP.
- **Force:** newton.
- **Current:** ampere + prefixes.
- Additions: volt, ohm, and lbf

### Arithmetic rules

- `+` and `−` require equal dimensions, otherwise **no autocomplete at all** (`1kg + 5m =` is silent).
- `*` and `/` combine dimension vectors.
- `^` needs a dimensionless exponent, and fractional powers must give integer dimension exponents (`sqrt(4 m^2)` = 2 m).
- Transcendental functions require dimensionless input, except trig, which takes angle or bare numbers.

Which units may take SI prefixes? (this resolves most symbol clashes such as `mi`/`min`/`pt`/`yd`/`ft`/`ha`/`cd`)
Only SI-style units take prefixes (g, m, s, Hz, J, Wh, cal, Pa, bar, L, W, A, V, N, Ω, B/b). Imperial, time-above-second, PSI, atm, etc. take none. Then `mi` is unambiguously miles, `min` is minutes, and `pt` is pint.

Bits/bytes prefixes: Large prefixes only (k through Q), in SI (1000) steps. `mb` isn't "millibit".

Regional and definitional variants:
Gallon/pint/quart/cup/fl oz: US customary. A `imp` or `uk` prefix selects imperial (`imp gal`).
Calorie: `cal` = 4.184 J (thermochemical). `kcal` = 1000 cal. `Cal` = kcal (food).
Year: 365.25 days (Julian). Decade is 10 years and century is 100.
HP: Show both as chips.
Knot: nautical mile per hour (1.852 km/h).

Derived unit naming (`3A*2V` → `6W`): A lookup table of known combinations is applied only when the product or quotient of *different* units exactly matches a named derived unit (A·V → W, W·s → J, J/s → W, N/m² → Pa, 1/s → Hz, …). Otherwise the result stays compound (`12mi^2`, `kg·m/s²`). Also adds volt and ohm.

Exponent notation for units: Accept `m^2`, `m**2`, `m²`, `m2`, `sq m`, `square meter`, `meters squared`, `cubic meter`, `m³`. Output uses `^` (`12mi^2`)

Temperature: Conversion only. A lone `20°C` converts to °F and K (affine offsets applied). `20°C + 5°C` is treated as adding a delta, result `25°C`. Multiplying or dividing absolute temperatures is disallowed.
Allow the prefix on kelvin (`mK`)

Unit plus bare number (`5m + 3`, `5m * 3`): `+`/`−` invalid (silent), `*`/`/` fine.

Unit-name output style: Symbols in suggestions (`9.001kg`). Long names appear only when the user typed a long name (mirroring: `5 kilometers` → `3.11 miles`).

---

## 8. Suggestion ordering and the bar

**Chip order:**
1. Primary result in the chosen unit.
2. Alternative units of the same dimension, from a curated per-dimension "popular units" list (e.g. for speed: mph, kph, ft/s, knot).
3. Longer decimals of the primary (bar only, never inline).
4. For trig, rad first, then deg, each with its own ladder.

The inline ghost is always chip 1. Chips marked "bar-only" never become the ghost.

Which unit is the primary for mixed-unit results (`1g + 9kg` → `9.001kg` first, `9001g` second): The unit giving a value in [1, 1000) (auto-prefix) first, then the operands' units, then other popular units. `1g+9kg` → kg, then g.

How many chips, and overflow?: Horizontally scrollable strip.

"[in other units]" content (e.g. `12mi^2`): A curated list per dimension (area: km², acre, ft², m², …).

Explicit conversion syntax: Also support `5 km to mi`, `5 km in miles`, `5km -> mi`, and autocomplete the target unit after `to` / `in`.

Variables and history: Not supported. Each expression is standalone.

---

## 9. Keyboard and mouse

- **Tab** accepts the inline (first) suggestion.
- **→** at end of line also accepts it.
- **Esc** dismisses until the next keystroke.
- Clicking a chip accepts it. `mousedown` calls `preventDefault` so the textarea keeps focus.
- After accepting, the caret goes to the end of the inserted text, and the suggestion engine runs again so chips chain (accept `=0.01`, then the bar offers `03` longer versions).
- Accepting never re-triggers a fresh "=" suggestion on the result.
- No suggestion appears while an IME composition is active.

Accepting non-first chips by keyboard: `Alt+1…9` accepts chip *n*, and `Alt+←/→` or `↑/↓` moves a highlight.

Q36 – Mobile and touch: Chips are tappable, with a big enough hit area. No Tab key is required.

---

## 10. Safety and edge cases

- Expression length cap, exponent and factorial magnitude caps, and a recursion depth cap, so the page can't hang.
- All evaluation is a hand-written parser. **No `eval`.**
- Evaluation is debounced (~30 ms) and skipped on IME composition and on selection ranges.
- Ghost text is removed when the caret moves away from the line end or when text is selected.
- Unicode normalization: `×`, `÷`, `−`, `π`, `µ` (micro sign) and `μ` (Greek mu) all work. `u` is an accepted alias for micro.
- Scientific notation input (`1.5e3`) is supported.

---

## 11. Acceptance tests

| Input | Expected |
|---|---|
| `1+1` | ghost `=2`, nothing longer |
| `1*5+9=` | ghost `14` |
| `1/97` | ghost `=0.01`, bar `=0.0103` `=0.010309` … |
| `1/97=0.01` | no ghost, bar shows `03`, … |
| `sin pi` | `=0` (rad), `=0.055` (deg) |
| `sin 90 =` | `0.9`, then `1` |
| `tan(sin(3*log5(3)))**(sqrt(0.3) + sin 2pi)` | rad value first, deg value second |
| `1g + 9kg =` | `9.001kg`, `9001g` |
| `1kg + 5m =` | nothing |
| `(5m + 16.4ft)/(0.04min + 2.6s) + 1 m/s =` | `=3m/s`, then mph, kph, … |
| `4mi * 3mi =` | `12mi^2`, then km², acre, ft², … |
| `3A * 2V =` | `6W`, then kW, HP, … |
| `3 PSI * 1in^2 =` | `3lbf`, then N |
