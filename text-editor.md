My Tests:
1+1
5(5)
11.7 + 79.9
sin(90)
sin(sin(sin(sin(sin(100))))) = -0.43
1g + 2g
1g + 1kg
67PSI * 4in^2 = 1192 N
(3m + 1km)/(1s + 1min/6) + 1m/s
408-555-1234
7/3/2026
- 5+5

Build a complete single-file HTML/CSS/JavaScript implementation using the following instructions. CDN dependencies are acceptable.

## 1. Shell and layout

- One `<textarea>` fills 100vw × 100vh with no border, outline, title, or toolbar. Only a thin **bottom bar** exists.
- The bar is a horizontally scrollable strip of suggestion chips. It is empty when there are none.
- When suggestions exist, the bar also shows a faint key hint: `Tab/Enter accept · ←→ switch`.
- Inline ghost text is drawn by a **mirror `<div>`** layered under or over a transparent-background textarea.
  - It copies font, padding, wrapping, and scroll position.
  - It contains the text before the caret, then a grey `<span>` with the ghost text.
- Insertion uses `document.execCommand('insertText')` so native undo/redo keeps working. `setRangeText` is the fallback.
- The theme follows `prefers-color-scheme`. The font is monospace.
- Content autosaves to `localStorage`.
- **Tab** with no acceptable suggestion inserts a real tab. **Enter** with no acceptable suggestion inserts a newline. "Acceptable" is defined in §9.

---

## 2. Architecture (all in one `<script>`)

1. **Line analyzer** takes the current line up to the caret and does four things:
   - strips the bullet marker;
   - splits on `=`;
   - finds the longest syntactically valid suffix;
   - classifies the result as inline, chip-only, or none.
2. **Tokenizer** handles numbers, operators, identifiers, unit tokens, and implicit multiplication. It also tags the `Ne-N` ambiguity (§4).
3. **Parser** is a precedence-climbing parser that produces an AST. It has no `eval`.
4. **Evaluator** works on `Quantity {value (Decimal), dims[], unit info}`.
   - Plain numbers are dimensionless quantities.
   - It runs in radians mode and degrees mode only if a bare-number trig argument exists.
   - **There is no complex-number support.**
5. **Unit engine** holds the unit table, prefix rules, dimension arithmetic, derived-unit naming, and the case-folded fallback resolver (§7).
6. **Formatter** handles exactness detection, the inline approximation rule, and the longer-decimal ladder.
7. **Suggestion engine** builds an ordered list of chips: `{insertText, displayEquation, typedPrefixLen, tag?, inlineEligible, kind}`.
8. **UI** covers the ghost, the chips, the highlight, key handling, and click handling.

---

## 3. Finding the expression and deciding when to fire

### 3.1 Preconditions
Nothing fires unless the caret is at the end of a line, or followed only by whitespace. Nothing fires when text is selected or during IME composition.

### 3.2 Pipeline (per keystroke, debounced ~30 ms)

1. **Take the line text before the caret.** Remember any trailing whitespace.
2. **Bullet stripping.**
   - If the line starts with optional indent, then a marker (`-`, `*`, `+`, `•`, `–`), then whitespace, the marker is a bullet and not a unary operator.
   - The remaining text is analyzed and the line is flagged `bullet`.
   - `-5+5` has no space after the marker, so it is not a bullet. It is unary minus and inlines normally.
3. **Segment on `=`.** Segments are separated by `=`, and earlier segments are never validated.
   - **Line ends with `=`:** the expression comes from the segment between the previous `=` (or line start) and that trailing `=`. This is the "`=` typed, empty result" shape.
     - Example: `x = 1+1 =` → expression `1+1`.
     - Example: `2+2 = 9+1 =` → expression `9+1`. The false `2+2 = 9+1` is ignored, so invalid math earlier on the line doesn't matter.
   - **Text after the last `=` is a partial result** (a number, optionally with a unit, such as `0.01` or `5 m`) **and the previous segment has a valid expression:** this is the "typed partial result" shape (§3.4).
   - **Otherwise** the text after the last `=` is itself the expression segment. This is the "no `=` typed" shape, and the suggestion inserts `=result`.
     - Example: `2+2 = 9+1` → expression `9+1`, ghost `=10`.
4. **Find the expression: longest syntactically valid suffix.**
   - Tokenize the segment. Try start positions from leftmost to rightmost. The first token run that parses completely, and ends exactly at the caret, wins.
   - Only tokens and grammar count. Unknown words are not valid tokens, so prose like "Total is" is skipped automatically. Two adjacent bare numbers (`3 4+5`) are not valid implicit multiplication.
   - **Once a suffix is chosen, evaluate it.** A semantic failure (dimension mismatch, domain error, overflow) is **silent, with no fallback to a shorter suffix.**
     - `1kg + 5m =` is silent.
     - `2*5m + 3kg =` is silent.
   - Caps apply: max 500 chars and max 200 tokens scanned.
5. **Classify the chosen expression.**

| Class | Rule | Behavior |
|---|---|---|
| **Lone literal** | A single number (optionally signed or parenthesized), e.g. `1`, `-5`, `(5)` | **Nothing at all**, neither ghost nor chips |
| **Constant** | `pi`, `π`, `e` | Inline normally |
| **Phone / date** | The whole expression matches a phone or date pattern (list below) | **Chip only** |
| **Bullet** | The expression starts immediately after a bullet marker (`- 5+5`) | **Chip only** |
| **Bare unit quantity** | e.g. `5km` | Chips with conversions, no ghost |
| **Ambiguous lowercase unit** | See §7 | Chip only |
| Anything else | | Inline |

   - Phone/date patterns, tested on the whole chosen expression:
     - `\d{3}-\d{4}`
     - `\d{3}-\d{3}-\d{4}`
     - `\(?\d{3}\)?[ -]?\d{3}-\d{4}`
     - `\d{4}-\d{1,2}-\d{1,2}`
     - `\d{1,2}[/.-]\d{1,2}[/.-]\d{2,4}`
   - If the user has typed an explicit `=`, the phone/date/bullet suppression is lifted and the ghost is allowed (see question 6).
6. **Evaluate** and build chips (§8).

### 3.3 Insertion shapes
- **No `=` typed:** insert `=result`. `1+1` → ghost `=2`.
- **`=` typed:** insert only the remainder. `1*5+9=` → ghost `14`.
- **Spacing mirrors the user.**
  - `1 + 1` → ` = 2`, and `1+1` → `=2`.
  - Units follow the user's number/unit spacing (`5m` vs `5 m`).
  - Output uses symbols, unless the user typed long names (`5 kilometers` → `3.11 miles`).

### 3.4 Typed partial results
- Typed text is matched as a prefix of each chip's result. Only the remainder is inserted (`=0.01` → chips offer `03`, …).
- If the typed result matches nothing (`1/97=0.02`), a **replace chip** swaps in the correct value. It is chip-only and never inline.
- A complete, matching result produces nothing. This also stops accepting `=2` from re-triggering another suggestion.

### 3.5 Name autocomplete
This is a separate path. When the caret follows a partial identifier, suggest completions for functions, constants, units, and conversion targets. For example, `sq` → ghost `rt(`, and `5 kilo` → `gram`/`meter`/…. Conversion targets are offered after `to`, `in`, and `->`.

---

## 4. Math grammar and precedence

- **Operators:** `+ - * / % ^ ** !`, parentheses, and `×`, `÷`, `−`, `π`, `µ`/`μ`. `^` and `**` are the same right-associative operator.
- **Precedence** (tightest to loosest):
  1. `!`
  2. `^` / `**`
  3. unary minus
  4. implicit multiplication
  5. infix `nPr` / `nCr`
  6. `* / %`
  7. `+ -`
- `-2^2` = −4. `2^-1` is allowed. `1/2pi` = 1/(2π).
- **Functions:**
  - `log` (base 10), `ln`, `log2`, `log_2`, `log5(3)`, `sqrt`, `cbrt`.
  - `sin cos tan csc sec cot`.
  - Inverses: `arcsin`, `asin`, `sin-1`, `sin^-1`, `sin⁻¹`.
- **Function application without parentheses:** the argument is the following implicit-multiplication run. `sin 2pi` = sin(2π), but `sin 2 * 3` = (sin 2)·3.
- **`sin-1` is the inverse only** when directly followed by `(` or a number with no space (`sin-1(0.5)`, `sin-1 0.5`). `sin -1` and `sin(-1)` mean sine of −1.
- **Trig powers:** `sin^2`, `sin**2`, `sin²`, `cos^2`, `tan^2`, and so on are supported, meaning (sin x)². The `sin2` form is pending (question 1).
- **`%`:**
  - `%` is **infix modulo** when the next token can start an operand: a number, constant, function, unit, or `(`.
  - It is **postfix percent** (÷100) otherwise. Postfix cases are end of expression, `)`, or a following `+`, `-`, `*`, `/`.
  - Examples: `10%3` = 1, `10%(3)` = 1, `50%` = 0.5, `50%)`, `50%+1` = 1.5.
  - Modulo needs dimensionless operands or equal dimensions.
- **Factorial:** non-negative integers up to 170, and non-integers via gamma. Anything else is silent.
- **nPr / nCr:** `nPr(5,2)`, `nCr(5,2)`, `5 nPr 2`, `5 nCr 2`, `P(5,2)`, `C(5,2)`.
- **Constants:** `e`, `pi`, `π`. `1e5` is scientific notation and `2e` is 2·e. `1.5e3` is supported.
- **`Ne-N` ambiguity** (e.g. `1e-2`): treated as scientific notation by default (see question 2).
- **No complex numbers.** These are all silent: `sqrt(-1)`, `(-8)^(1/3)`, `asin(2)`, `acos(2)`, `log(0)`, `log(-1)`, `0/0`, and overflow past the caps.

---

## 5. Radians and degrees

- If any trig function has a bare-number argument, evaluate the **whole expression** in both modes.
- **Chip order:** rad primary, deg primary, rad ladder, deg ladder.
- Each trig chip carries a faint `rad` or `deg` tag.
- Inverse trig outputs follow the same rule: `asin(0.5)` gives 0.52 (rad), then 30 (deg).
- Explicit angle units (`sin(30deg)`, `sin(30°)`, `sin(pi rad)`) produce a single result with no tag.
- `sin(30°) + cos(1)` has a bare `1`, so it is still ambiguous and produces two chips.
- **Per-mode failure:** if one mode errors or hits a pole (|tan| beyond ~1e15), that mode's chips are dropped and the other survives. If both fail, the result is silent.
- Duplicate chips are removed (`sin 0` shows one chip).
- Examples:
  - `sin pi` → `=0` (rad), `=0.055` (deg).
  - `sin 90` → `0.9` (rad), `1` (deg), then the rad ladder.

---

## 6. Number formatting and the decimal ladder

- **Arithmetic** uses Decimal.js at ~50 digits.
- **Cleaning:** values within ~1e-14 of zero or of an integer snap to it, so `sin(pi)` → 0.
- **Exactness test:** if the cleaned value is a terminating decimal with ≤ 20 significant digits, it is **exact**.
  - Exact values are shown **in full**: `1/8` → `=0.125`, `19.99*3` → `=59.97`.
  - Exact values never get ladder chips.
  - Everything else, including irrational results and non-terminating decimals, is **approximate**.
- **Inline approximation** (approximate values only). Use the fewest decimals *d* ≥ 1 such that all three hold:
  - relative error ≤ 5%;
  - the rounded value is non-zero;
  - the integer part is unchanged (no rounding into or away from integer digits).
  - Examples: `1/97` → `0.01`, `1/3` → `0.33`, `pi` → `3.1`, `e` → `2.7`, `100/3` → `33.3`, `sin 90°` → `0.9`, and `sin π°` → `0.055` (`0.05` is 8.8% off).
- **Longer ladder (bar only, never inline):** 3, 5, and 8 significant figures.
  - Entries are deduplicated, and anything no more precise than the inline value is dropped.
  - A step is skipped if it would round integer-part digits (for `1234.5678`, 3 sf is skipped).
  - `1/97` gives `0.0103`, `0.010309`, `0.010309278`. `pi` gives `3.14`, `3.1416`, `3.1415927`.
- **Alternate-unit chips** (conversions):
  - Exact values are shown in full.
  - Approximate values use 3 sf, never rounding integer digits.
  - Only the primary chip gets a ladder.
- **Scientific notation** is used for values ≥ 1e15 or < 1e-6. The mantissa follows the same rules.
- Inline approximations are not marked. The ghost is plain `=0.01`.

---

## 7. Units engine

### Design
- Each unit is stored as `{names[], symbols[], dims, factorToSI, offset?, prefixable: 'all' | 'large' | 'none'}`.
- Dimension vector: length, mass, time, current, temperature, data, angle.
- Symbols are **case-sensitive**. Full names are case-insensitive, with plurals and alternate spellings.
- Prefix parsing is greedy, and an **exact base-unit match beats prefix+unit**.
- Unit names and symbols autocomplete as you type.

### Symbol resolution order (lowercase leniency)
1. **Exact case-sensitive match.** This is unambiguous and inline-eligible.
2. **Only if step 1 fails, a case-folded fallback** enumerates all (prefix case × symbol case) candidates.
   - **Exactly one candidate** is accepted and inline-eligible. Output uses the canonical symbol, so `5khz` becomes `kHz`.
   - **Multiple candidates** make the suggestion **chip-only**, with one chip per reading.
     - The reading that preserves the typed case comes first.
     - Each chip's displayed equation shows its reading. For `5mw + 1w =`, one chip is `5mW + 1W = 1.005W` and another is `5MW + 1W = 5.000001MW`.
   - Another example: `mb` could be Mb or MB, so it is chip-only.

### Coverage
- **Mass:** gram + all prefixes, ounce, pound.
- **Area:** square foot/meter/kilometer/mile/yard/inch, acre, `ft^2`, `ft²`, `sq ft`.
- **Data:** bit and byte with large prefixes only (k through Q, 1000 steps), optionally per second (`Mbps`, `MB/s`).
- **Energy:** joule + prefixes, calorie, watt-hour + prefixes.
- **Frequency:** hertz + prefixes.
- **Fuel economy:** mpg, km/L.
- **Length:** mile, yard, foot, inch, meter + prefixes.
- **Angle:** degree, radian, gradian.
- **Pressure:** pascal + prefixes, bar + prefixes, atm, PSI.
- **Speed:** mph, ft/s, m/s, kph, knot (1.852 km/h).
- **Temperature:** °F, °C, K (K takes prefixes).
- **Time:** second + prefixes, minute, hour, day, week, month, year, decade, century.
- **Volume:** gallon, pint, quart, cup, tbsp, tsp, m³, liter + prefixes, ft³, in³, fl oz.
- **Power:** watt + prefixes, HP (show both HP variants as chips).
- **Force:** newton, lbf.
- **Current:** ampere + prefixes.
- **Also:** volt, ohm.
- **Prefixable units:** only SI-style units (g, m, s, Hz, J, Wh, cal, Pa, bar, L, W, A, V, N, Ω, B/b, K). Imperial units, time above seconds, PSI, and atm take no prefixes. So `mi` is miles, `min` is minutes, and `pt` is pint.

### Variants
- **US customary** for gallon, pint, quart, cup, and fl oz. `imp` or `uk` selects imperial.
- **Calorie:** `cal` = 4.184 J, `kcal` = 1000 cal, `Cal` = kcal.
- **Year:** 365.25 days. Decade is 10 years, century is 100.
- **µ/μ/u** are all accepted as micro.

### Arithmetic rules
- `+` and `−` need equal dimensions, otherwise the result is silent. A unit plus a bare number is also silent.
- `*` and `/` combine dimensions, and a bare number with `*` or `/` is fine.
- `^` needs a dimensionless exponent. Fractional powers must give integer dimension exponents (`sqrt(4 m^2)` = 2 m).
- Transcendental functions need dimensionless input. Trig accepts angle units or bare numbers.
- **Temperature:**
  - A lone `20°C` converts to °F and K.
  - `20°C + 5°C` is a delta, giving `25°C`.
  - `*` and `/` on absolute temperatures are silent.

### Derived unit naming
A lookup table applies only when the product or quotient of *different* units exactly matches a named unit. Examples: A·V → W, W·s → J, J/s → W, N/m² → Pa, 1/s → Hz, PSI·in² → lbf. Otherwise the result stays compound (`12mi^2`, `kg·m/s²`).

### Exponent notation
Input accepts `m^2`, `m**2`, `m²`, `m2`, `sq m`, `square meter`, `meters squared`, `cubic meter`, `m³`. Output uses `^` (`12mi^2`).

### Explicit conversion
`5 km to mi`, `5 km in miles`, `5km -> mi`.

---

## 8. Suggestion ordering and the bar

### Chip anatomy
Each chip shows the **full formatted equation**, `expression = result`.
- The part the user already typed is dim, and the part that would be inserted is emphasized. For `1/97=0.01`, the chip shows `1/97 = 0.01` with `03` highlighted.
- Trig chips carry a faint `rad` or `deg` tag.
- Long equations are ellipsized in the middle inside the scrolling strip.
- Clicking a chip inserts only `insertText`, never the equation.
- Exact display format for the equation is pending (question 2).

### Order
1. Primary result in the chosen unit.
2. Alternative units from a curated per-dimension list. Examples: speed gives mph, kph, ft/s, knot. Area gives km², acre, ft², m².
3. Longer decimals of the primary.
4. For trig: rad primary, deg primary, rad ladder, deg ladder.

**Primary unit for mixed units:** the unit whose value falls in [1, 1000) comes first, then the operands' units, then other popular units. `1g + 9kg` gives `9.001kg`, then `9001g`.

### Inline eligibility
A chip is **inline-eligible** only if all of these hold:
- It is a primary: the main result, a trig mode primary, or the remainder of a typed partial result.
- The line class is *inline* (§3.2).
- There is no lowercase-unit ambiguity.

These are **bar-only**: the decimal ladder, alternate-unit chips, replace chips, ambiguous-reading chips, and everything on phone, date, bullet, and bare-unit lines.

### Ghost and highlight
- The highlight starts on chip 1.
- **The ghost previews the highlighted chip if it is inline-eligible, otherwise no ghost.**

### Other
- A bare unit (`5km`) shows conversions in the bar only. Insertion is `=3.11mi` and so on.
- No variables or history. Each expression is standalone.

---

## 9. Keyboard and mouse

- **Tab and Enter** accept the highlighted chip, but only when it is *acceptable*:
  - a ghost is visible, or
  - the user has explicitly moved the highlight with the arrows.
  - Otherwise Tab inserts a tab and Enter inserts a newline.
  - This keeps phone numbers and dates from hijacking Enter.
- **Arrow keys switch chips** while chips exist:
  - `←` and `↑` go to the previous chip, and `→` and `↓` go to the next.
  - At either end of the list, the key falls through to normal caret movement, which dismisses the ghost.
  - `→` no longer accepts the ghost.
- **Alt+1…9** accepts chip *n* directly.
- **Esc** dismisses until the next keystroke.
- **Click or tap** accepts a chip. `mousedown` calls `preventDefault` so focus stays in the textarea. Touch targets are large, and no Tab key is needed.
- After accepting, the caret goes to the end of the inserted text and the engine re-runs so chips chain. Accepting `=0.01` makes the bar offer `03`, and so on. Accepting never triggers a fresh `=` suggestion on the result.
- No suggestions appear during IME composition.

---

## 10. Safety and edge cases

- Caps apply to expression length, exponent magnitude, factorial magnitude, recursion depth, and tokens scanned.
- No `eval`. Evaluation is debounced (~30 ms) and skipped on composition and on selection ranges.
- The ghost is removed when the caret leaves the line end or text is selected.
- Unicode normalization covers `×`, `÷`, `−`, `π`, and `µ`/`μ`. `u` is an alias for micro.
- Trig poles (|result| beyond ~1e15) count as undefined.

---

## 11. Acceptance tests

| Input | Expected |
|---|---|
| `1+1` | ghost `=2`, no ladder. Bar: `1+1 = 2` |
| `1*5+9=` | ghost `14` |
| `1/97` | ghost `=0.01`. Bar: `=0.01`, `=0.0103`, `=0.010309`, `=0.010309278` |
| `1/97=0.01` | no ghost. Bar: `03`, `0309`, `0309278` highlighted |
| `1/97=0.02` | no ghost. Bar: replace chip with `0.01` |
| `1/8` | ghost `=0.125`, no ladder |
| `19.99*3` | ghost `=59.97` |
| `1/3` | ghost `=0.33`. Ladder `0.333`, `0.33333`, `0.33333333` |
| `pi` | ghost `=3.1`. Ladder `3.14`, `3.1416`, `3.1415927` |
| `e` | ghost `=2.7` |
| `1` | nothing |
| `sin pi` | `=0` (rad), `=0.055` (deg) |
| `sin 90 =` | `0.9` (rad), `1` (deg), then the rad ladder |
| `tan(pi/2)` | rad hits a pole and is dropped. Single chip `=0.027` (deg) |
| `tan(sin(3*log5(3)))**(sqrt(0.3) + sin 2pi)` | rad value first, deg value second |
| `sqrt(-1)`, `log(0)`, `asin(2)`, `0/0` | nothing |
| `10%3`, `10%(3)` | `=1` |
| `50%` | exception; do nothing |
| `50%+1` | `=1.5` |
| `sin^2(30)` | rad chip, deg chip, each tagged |
| `x = 1+1 =` | ghost `2` |
| `2+2 = 9+1 =` | ghost `10` |
| `2+2 = 9+1` | ghost `=10` |
| `555-1234` | no ghost. Chip `555-1234 = -679` |
| `2024-01-15` | no ghost. Chip `= 2008` |
| `- 5+5` | no ghost. Chip `5+5 = 10` |
| `-5+5` | ghost `=0` |
| `1g + 9kg =` | `9.001kg`, `9001g` |
| `1kg + 5m =` | nothing |
| `2*5m + 3kg =` | nothing |
| `5km` | no ghost. Bar: `=3.11mi`, `=5000m`, … |
| `5mw + 1w =` | no ghost. Two chips: mW reading first, MW reading second |
| `3kw * 2s =` | ghost `6kJ` |
| `(5m + 16.4ft)/(0.04min + 2.6s) + 1 m/s =` | `=3m/s`, then mph, kph, … |
| `4mi * 3mi =` | `12mi^2`, then km², acre, ft², … |
| `3A * 2V =` | `6W`, then kW, HP, … |
| `3 PSI * 1in^2 =` | `3lbf`, then N |

---

## 12. Remaining ambiguities

A digit run glued to the function name is a power when an argument follows (`sin2(30)`, `sin2 x`, `sin2 pi`). A bare `sin2` with nothing after it means sin(2).

When a token pattern like `Ne-N` is genuinely ambiguous, show two chips, scientific notation first, each with its equation rendered as parsed (`1 + 10⁻² = 1.01` and `1 + 1·e − 2 = 1.72`). Everything else shows a single equation in canonical spacing.

Approximate values always keep ≥ 1 decimal and the integer part is never altered. So `pi` → `3.1`, `100/3` → `33.3`, and `9.96…` → `9.96`.

Enter only accepts when a ghost is visible or you've explicitly moved the highlight. Otherwise it's a newline, so a phone number line never eats Enter.
`←`/`↑` and `→`/`↓` switch chips, falling through to the caret at the ends of the list, and `→` no longer accepts.

lowercase prefixe:
 - One possible reading is accepted and inlined.
 - Several readings (`mw` → mW or MW) make it chip-only, one chip per reading.
 - Exact-case matches always win and are never treated as ambiguous.

When the user types an explicit `=`, should the no-inline rules lift?
Yes, so `555-1234=` ghosts `-679` and `- 5+5 =` ghosts `10`. Typing `=` signals intent. A lone number like `1=` still shows nothing.

Other things
- Bullet markers are `- * + • –`.
- Infix `nPr`/`nCr` bind tighter than `* /`.
- Modulo requires dimensionless or equal-dimension operands.
- "Exact" means terminating with ≤ 20 significant digits.
- Scientific notation applies to exact values too (≥ 1e15 or < 1e-6).
- One exception: a lone `50%` should not fire.
