Build a complete HTML/CSS/JavaScript implementation using the following instructions. CDN dependencies are acceptable.

## Shell and layout

- One `<textarea>` fills 100vw × 100vh with no border, outline, title, or toolbar. Only a thin **bottom bar** exists.
- The bar is a horizontally scrollable strip of suggestion chips. It is empty when there are none.
- When suggestions exist, the bar also shows a faint key hint: `Tab to accept · ←→ switch`.
- Inline ghost text is drawn on a transparent-background textarea.
  - It copies font, padding, wrapping, and scroll position.
  - It contains the text before the caret, then gray ghost text.
- Allow for undo/redo via native commands.
- The theme follows `prefers-color-scheme`. The font is monospace.
- Content autosaves to `localStorage`.
- Tab with autocompletes both inline and chip suggestions. Tab with no suggestion inserts a real tab. Enter never autocorrects and always inserts a newline.

---

## Finding the expression and deciding when to fire

### Preconditions

Nothing fires unless the caret is at the end of a line, or followed only by whitespace. Nothing fires when text is selected or during IME composition.

1. **Take the line text before the caret.** Remember any trailing whitespace.
2. **Bullet stripping.**
   - If the line starts with optional indent, then a marker (`-`, `*`, `+`, `•`, `–`), then whitespace, the marker is a bullet and not a unary operator.
   - The remaining text is analyzed and the line is flagged `bullet`.
   - `-5+5` has no space after the marker, so it is not a bullet. It is unary minus and inlines normally.
3. **Segment on `=`.** Segments are separated by `=`, and earlier segments are never validated.
   - **Line ends with `=`:** the expression comes from the segment between the previous `=` (or line start) and that trailing `=`. This is the "`=` typed, empty result" shape.
     - Example: `x = 1+1 =` → expression `1+1`.
     - Example: `2+2 = 9+1 =` → expression `9+1`. The false `2+2 = 9+1` is ignored, so invalid math earlier on the line doesn't matter.
   - **Text after the last `=` is a partial result** (a number, optionally with a unit, such as `0.01` or `5 m`) **and the previous segment has a valid expression:**
   - **Otherwise** the text after the last `=` is itself the expression segment. This is the "no `=` typed" shape, and the suggestion inserts `=result`.
     - Example: `2+2 = 9+1` → expression `9+1`, ghost `=10`.
4. **Find the expression: longest syntactically valid suffix.**
   - Tokenize the segment. Try start positions from leftmost to rightmost. The first token run that parses completely, and ends exactly at the caret, wins.
   - Only tokens and grammar count. Unknown words are not valid tokens, so prose like "Total is" is skipped automatically. Two adjacent bare numbers (`3 4+5`) are not valid implicit multiplication.
   - **Once a suffix is chosen, evaluate it.** A semantic failure (dimension mismatch, domain error, overflow) is **silent, with no fallback to a shorter suffix.**
     - `1kg + 5m =` is silent.
     - `2*5m + 3kg =` is silent.
5. **Classify the chosen expression.**

| Class | Rule | Behavior |
|---|---|---|
| **Lone literal** | A single number (optionally signed or parenthesized), e.g. `1`, `-5`, `(5)` | **Nothing at all**, neither ghost nor chips |
| **Constant** | `pi`, `π`, `e` | Inline normally |
| **Phone / date** | The whole expression matches a phone or date pattern (list below) | **Chip only** |
| **Bullet** | The expression starts immediately after a bullet marker (`- 5+5`) | **Chip only** |
| **Bare unit quantity** | e.g. `5km` | Chips with conversions, no ghost |
| **Ambiguous lowercase unit** | See unit symbol resolution | Chip only |
| Anything else | | Inline |

- Phone/date patterns, tested on the whole chosen expression:
  - `\d{3}-\d{4}`
  - `\d{3}-\d{3}-\d{4}`
  - `\(?\d{3}\)?[ -]?\d{3}-\d{4}`
  - `\d{4}-\d{1,2}-\d{1,2}`
  - `\d{1,2}[/.-]\d{1,2}[/.-]\d{2,4}`
- If the user has typed an explicit `=`, the phone/date/bullet suppression is lifted and the ghost is allowed.
6. **Evaluate** and build chips.

### Insertion shapes

- **No `=` typed:** insert `=result`. `1+1` → ghost `=2`.
- **`=` typed:** insert only the remainder. `1*5+9=` → ghost `14`.
- **Spacing mirrors the user.**
  - `1 + 1` → ` = 2`, and `1+1` → `=2`.
  - Units follow the user's number/unit spacing (`5m` vs `5 m`).
  - Output uses symbols, unless the user typed long names (`5 kilometers` → `3.11 miles`).

### Typed partial results

- Typed text is matched as a prefix of each chip's result. Only the remainder is inserted (`=0.01` → chips offer `03`, …).
- If the typed result matches nothing (`1/97=0.02`), a **replace chip** swaps in the correct value. It is chip-only and never inline.
- A complete, matching result produces nothing. This also stops accepting `=2` from re-triggering another suggestion.

### Name autocomplete

This is a separate path. When the caret follows a partial identifier, suggest completions for constants, units, and conversion targets. For example, `cel` → ghost `cius`, and `5 kilo` → `gram`/`meter`/…. Conversion targets are only offered after `to`, `in`, and `->`.

---

## Math grammar and precedence

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
  - Inverses: `arcsin`, `asin`, `sin-1`, `sin^-1`.
- **Function application without parentheses:** the argument is the following implicit-multiplication run. `sin 2pi` = sin(2π), but `sin 2 * 3` = (sin 2)·3.
- **`sin-1` is the inverse only** when directly followed by `(` or a number with no space (`sin-1(0.5)`, `sin-1 0.5`). `sin -1` and `sin(-1)` mean sine of −1.
- **Trig powers:** `sin2`, `sin^2`, `sin**2`, `cos^2`, `tan^2`, and so on are supported, meaning (sin x)².
- **`%`:**
  - `%` is **infix modulo** when the next token can start an operand: a number, constant, function, unit, or `(`.
  - It is **postfix percent** (÷100) otherwise. Postfix cases are end of expression, `)`, or a following `+`, `-`, `*`, `/`.
  - Examples: `10%3` = 1, `10%(3)` = 1, `50%` = 0.5, `50%+1` = 1.5.
  - Modulo needs dimensionless operands or equal dimensions.
- **Factorial:** non-negative integers up to 170, and non-integers via gamma. Anything else is silent.
- **nPr / nCr:** `nPr(5,2)`, `nCr(5,2)`, `5 nPr 2`, `5 nCr 2`, `P(5,2)`, `C(5,2)`.
- **Constants:** `e`, `pi`, `π`. `1e5` is scientific notation and `2e` is 2·e. `1.5e3` is supported.
- **`Ne-N` ambiguity** (e.g. `1e-2`): treated as scientific notation by default.
- **No complex numbers.** These are all silent: `sqrt(-1)`, `(-8)^(1/3)`, `asin(2)`, `acos(2)`, `log(0)`, `log(-1)`, `0/0`, and overflow past the caps.

---

## Radians and degrees

- If any trig function has a bare-number argument, evaluate the **whole expression** in both modes.
- **Chip order:** rad primary, deg primary, rad ladder, deg ladder.
- Each trig chip carries a faint `rad` or `deg` tag.
- Inverse trig outputs follow the same rule: `asin(0.5)` gives 0.52 (rad), then 30 (deg).
- Explicit angle units (`sin(30deg)`, `sin(30°)`, `sin(pi rad)`) produce a single result with no tag.
- `sin(30 deg) + cos(1)` has a bare `1`, so it is still ambiguous and produces two chips.
- **Per-mode failure:** if one mode errors or hits a pole (|tan| beyond ~1e15), that mode's chips are dropped and the other survives. If both fail, the result is silent.
- Duplicate chips are removed (`sin 0` shows one chip).
- Examples:
  - `sin pi` → `=0` (rad), `=0.055` (deg).
  - `sin 90` → `0.9` (rad), `1` (deg), then the rad ladder.

---

## Number formatting and the decimal ladder

- **Arithmetic** uses Decimal.js.
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

## Units engine

### Core design: one comprehensive unit database, seven SI base dimensions

Build a large, extensible `UNIT_DATABASE`. Every unit record must contain all seven exponent fields explicitly, including zeros:

```js
{
  name: "newton",
  coefficient: 1,
  m: 1,
  kg: 1,
  s: -2,
  A: 0,
  K: 0,
  mol: 0,
  cd: 0
}
```

- `coefficient` is the multiplicative factor that converts one unit into coherent SI base units. For ordinary non-offset units, `SI magnitude = displayed magnitude * coefficient`.
- The seven integer exponents (`m`, `kg`, `s`, `A`, `K`, `mol`, `cd`) are the unit's complete physical dimension vector. Normalize and compare dimensions as a seven-element vector in this fixed order: `m, kg, s, A, K, mol, cd`.
- Add useful non-dimensional metadata such as `symbols[]`, `aliases[]`, `pluralNames[]`, `prefixable`, `displayPriority`, `kind`, and `offset` when needed for parsing, display, affine temperature conversion, or semantic distinctions. These fields must not replace or alter the seven base exponents.
- The database should contain extensive coverage, not just a short hard-coded whitelist. Include SI base units, SI derived units, common non-SI units, imperial and US customary units, pressure units, common compound-unit spellings, aliases, and prefix variants. Generate valid prefix variants systematically at initialization or store them as generated database entries. Do not maintain a separate table mapping pairs or combinations of units to results.
- All unit calculations must be based on `coefficient` and the seven exponents. Never make a unit operation fail merely because the resulting dimensions do not have a familiar SI name.

### Required database examples

Use these records (with aliases and symbols as appropriate) as schema and value examples:

```js
{ name: "meter", coefficient: 1, m: 1 }
{ name: "kilogram", coefficient: 1, kg: 1 }
{ name: "second", coefficient: 1, s: 1 }
{ name: "ampere", coefficient: 1, A: 1 }
{ name: "kelvin", coefficient: 1, K: 1 }
{ name: "mole", coefficient: 1, mol: 1 }
{ name: "candela", coefficient: 1, cd: 1 }

{ name: "atmospheric pressure", symbols: ["atm"], aliases: ["standard atmosphere", "atmosphere"], coefficient: 101325, }
m: -1, kg: 1, s: -2, A: 0, K: 0, mol: 0, cd: 0

```

The examples above abbreviate omitted zero-valued fields for readability. The actual database must store all seven exponent fields on every record.

Additional examples:

- `gram`: coefficient `0.001`, `kg: 1`.
- `minute`: coefficient `60`, `s: 1`.
- `liter`: coefficient `0.001`, `m: 3`.
- `inch`: coefficient `0.0254`, `m: 1`.
- `pound-mass`: coefficient approximately `0.45359237`, `kg: 1`.
- `pound-force`: coefficient approximately `4.4482216152605`, `m: 1`, `kg: 1`, `s: -2`.

Use coefficients to represent prefixes and non-SI scale factors. For example, `gigafarad` is the farad coefficient multiplied by `10^9`, and `nanovolt` is the volt coefficient multiplied by `10^-9`. Prefix application changes the coefficient only; the seven exponents remain those of the underlying unit.

### Unit coverage

Populate the database comprehensively in at least these categories:

- **SI base and derived:** m, kg, s, A, K, mol, cd; g; Hz; N; Pa; J; W; C; V; Ω; F; H; S; Wb; T; lm; lx; and other widely used SI derived units such as Gy, Sv, katal, and Bq.
- **Mass:** gram and SI prefixes, tonne, ounce, pound, stone, and other common mass units.
- **Length:** meter and SI prefixes, kilometer, mile, yard, foot, inch, nautical mile, and other common length units.
- **Area and volume:** square/cubic forms of common length units, acre, liter and prefixes, US and imperial gallon/pint/quart/cup/fluid ounce, tablespoon, teaspoon, cubic foot, and cubic inch.
- **Time:** second and SI prefixes, minute, hour, day, week, year (365.25 days), decade, and century.
- **Pressure:** pascal and prefixes, bar and supported prefixes, standard atmosphere (`atm`, coefficient `101325`), torr, mmHg, inHg, and psi/PSI. Pressure records use dimensions `m^-1 kg s^-2`.
- **Energy and power:** joule and prefixes, calorie, kilocalorie/Calorie, watt-hour and prefixes, watt and prefixes, metric and mechanical horsepower variants.
- **Electrical:** ampere, coulomb, volt, ohm, farad, henry, siemens, weber, tesla, and common derived electrical units; support units such as mAh through standard multiplication/division of units.
- **Speed and acceleration:** m/s, km/h or kph, mph, ft/s, knot, and acceleration units.
- **Angle:** radian, degree, and gradian. These are dimensionless in the seven SI base dimensions; mark them with auxiliary `kind: "angle"` metadata so trigonometric behavior and angle display work correctly.
- **Data:** bit, byte, and common bit/byte multiples, optionally per second. They do not introduce an eighth SI base exponent. Use auxiliary `kind: "data"` / `kind: "data-rate"` metadata to prevent data quantities from being mislabeled as other dimensionless or inverse-time quantities and to restrict their display conversions to compatible units.
- **Fuel economy:** mpg and km/L, using their physical dimension vectors and scale coefficients.
- **Temperature:** K, °C, and °F. Since Celsius and Fahrenheit are affine units, support their scale and offsets explicitly; do not try to encode an offset using a coefficient alone.

### Prefixes, symbols, and name resolution

- Support every standard SI prefix and its exact decimal factor where applicable:
  - quecto `q` = 10^-30; ronto `r` = 10^-27; yocto `y` = 10^-24; zepto `z` = 10^-21; atto `a` = 10^-18; femto `f` = 10^-15; pico `p` = 10^-12; nano `n` = 10^-9.
  - micro `µ`/`μ`/`u` = 10^-6; milli `m` = 10^-3; centi `c` = 10^-2; deci `d` = 10^-1.
  - deca `da` = 10^1; hecto `h` = 10^2; kilo `k` = 10^3; mega `M` = 10^6; giga `G` = 10^9; tera `T` = 10^12; peta `P` = 10^15; exa `E` = 10^18; zetta `Z` = 10^21; yotta `Y` = 10^24; ronna `R` = 10^27; quetta `Q` = 10^30.
- Preserve case distinctions in symbols.
- By default, allow prefixes on suitable SI units such as m, g, s, Hz, J, Wh, cal, Pa, bar, L, W, A, V, N, Ω, F, H, S, Wb, T, K, and bit/byte. Configure prefixability on each record.
- Do not prefix units that conventionally do not take SI prefixes (for example, mile, minute, psi, and atm).
- Handle gram/kilogram correctly: `kg` is the SI base mass unit, while prefixing gram must produce the expected scale (`mg = 10^-6 kg`, `g = 10^-3 kg`, `kg = 1 kg`, `Mg = 10^3 kg`).
- An exact full unit or symbol match must beat a parsed prefix-plus-unit interpretation.
- Symbols are case-sensitive. Full names and aliases are case-insensitive and support sensible plurals and alternate spellings.
- Resolve symbols in this order:
  1. Exact case-sensitive symbol/name match.
  2. If that fails, enumerate case-folded prefix/symbol readings.
  3. If there is one candidate, accept it inline and output canonical capitalization.
  4. If there are multiple candidates, create chip-only suggestions, one per reading, with the reading that best preserves typed capitalization first. Example: `mw` may be mW or MW; it must not silently choose between them. `5mw + 1w =` presents each valid interpretation as a separate chip.
- Autocomplete names, symbols, aliases, and conversion targets from this database. Conversion targets are suggested only after `to`, `in`, or `->`.

### Dimension-vector arithmetic (mandatory)

Represent every evaluated quantity as a numeric SI-scaled magnitude plus its normalized dimension vector and any necessary semantic metadata. For multiplicative, non-offset units:

- **Multiply:** multiply numeric SI magnitudes and add the seven corresponding exponents.
- **Divide:** divide numeric SI magnitudes and subtract the seven corresponding exponents.
- **Raise to a power:** multiply each exponent by the dimensionless exponent. Permit fractional powers only when every resulting base exponent is an integer within the configured exponent cap; otherwise treat the expression as undefined.
- **Add/subtract:** require equal seven-component dimension vectors and compatible semantic kinds. Bare-number plus a dimensioned quantity remains invalid. Preserve the existing temperature rules for absolute temperatures and temperature deltas.
- **Remainder/modulo:** require dimensionless operands or equal dimensions, as already specified in the math grammar.
- **Functions:** transcendental functions require dimensionless input, except for explicitly supported angle quantities in trigonometric functions.
- Do not reject a valid multiplication, division, or power merely because its dimensions are unfamiliar or no named unit exists.

### Naming and formatting results without pair-combination tables

- **Never use an operand-pair lookup table** to decide what `unitA * unitB`, `unitA / unitB`, or a power should become. Determine the result by adding/subtracting/multiplying the seven exponents and calculating the SI-scaled magnitude.
- Build a direct index from each database unit's normalized dimension vector to the units that have that vector. This is a reverse index of individual unit records, not a table of unit combinations. When the result vector matches one or more known units, select the display unit by these priorities: preserve a sensible operand/user-selected unit when appropriate; otherwise choose a recognized named unit and prefix whose displayed magnitude is convenient (prefer values in `[1, 1000)`); break ties using a small per-record display priority. Do not create hand-written rules for every pair of units.
- If no named unit matches the resulting vector, **return and display the valid result rather than failing**. Display the numeric SI-scaled magnitude followed by the remaining base dimensions in this order: `m`, `kg`, `s`, `A`, `K`, `mol`, `cd`. Omit zero exponents; omit `^1`; use `^` for all other exponents; separate terms with spaces.
- Example: `3 m^-4 kg^-2 s^7 A^3`. If all exponents are zero, display the dimensionless number, unless auxiliary semantic metadata requires preserving a data/angle kind.
- When no named unit matches, do not fabricate a unit name or round away dimensions. When a named unit matches, divide the SI-scaled magnitude by that unit's coefficient and display the resulting value with the canonical symbol/name.
- Because matching is based on dimensions, `farad * ohm` automatically reduces to seconds; `ampere * volt` can display watts; `watt * second` can display joules; and `newton / meter^2` can display pascals, without any pair-specific rules.
- **Required example:** `3 gigafarads / 1 nanovolt / 10^18` evaluates successfully to `3 m^-4 kg^-2 s^7 A^3`. Farad has dimensions `m^-2 kg^-1 s^4 A^2`; volt has dimensions `m^2 kg s^-3 A^-1`; dividing the former by the latter gives `m^-4 kg^-2 s^7 A^3`, and the prefixes/numeric divisor leave coefficient `3`.

### Special conversions and temperature handling

- For explicit conversion (`5 km to mi`, `5 km in miles`, `5km -> mi`), require compatible dimensions and semantic kinds. Convert through the SI-scaled magnitude and the target unit's coefficient/offset.
- Common alternatives may be offered by querying the unit database for records with the same dimension vector; do not maintain another large table of pairwise conversions. Filter out incompatible semantic kinds and rank/cap suggestions for usability.
- **Temperature:** a lone `20°C` offers °F and K conversions. `20°C + 5°C` follows the existing temperature-delta behavior and gives `25°C`. Multiplication/division of absolute temperatures is silent. Implement Celsius/Fahrenheit offsets explicitly while performing arithmetic in kelvin or temperature deltas as appropriate.
- **Angle:** radians/degrees ambiguity and the existing dual-mode trigonometric suggestions remain as specified above. Angle units are recognized through the unit database and `kind` metadata, not an eighth base dimension.
- **Data:** preserve bit/byte and data-rate conversion behavior through `kind` metadata; do not label a data rate as Hz merely because its seven physical exponents match `s^-1`.
- **Variants:** support US customary units by default for gallon, pint, quart, cup, and fluid ounce; `imp` or `uk` selects imperial variants. `cal = 4.184 J`, `kcal = 1000 cal`, and `Cal = kcal`. A year is 365.25 days; a decade is 10 years; a century is 100 years. Show both horsepower definitions as distinct alternatives when they differ.

### Exponent notation

Input accepts `m^2`, `m**2`, `m²`, `m2`, `sq m`, `square meter`, `meters squared`, `cubic meter`, and `m³`. Output uses `^` (for example, `12mi^2` or `3 m^-4 kg^-2 s^7 A^3`).

### Explicit conversion

Support `5 km to mi`, `5 km in miles`, and `5km -> mi`, plus conversions among `torr`, `atm`, `Pa`, `psi`, `mmHg`, and `inHg`.

---

## 8. Suggestion ordering and the bar

### Chip anatomy

Each chip shows the **full formatted equation**, `expression = result`.

- The part the user already typed is dim, and the part that would be inserted is emphasized. For `1/97=0.01`, the chip shows `1/97 = 0.01` with `03` highlighted.
- Trig chips carry a faint `rad` or `deg` tag.
- Long equations are ellipsized in the middle inside the scrolling strip.
- Clicking a chip inserts only `insertText`, never the equation.
- Exact display format for the equation is pending.

### Order

1. Primary result in the chosen unit.
2. Alternative units found by querying the unit database for compatible units with the same seven-exponent dimension vector, ranked by display priority and convenient magnitude. Do not use a unit-pair conversion table. Examples: speed may offer mph, kph, ft/s, knot; area may offer km², acre, ft², m².
3. Longer decimals of the primary.
4. For trig: rad primary, deg primary, rad ladder, deg ladder.

**Primary unit for mixed units:** the unit whose value falls in [1, 1000) comes first, then the operands' units, then other popular units. `1g + 9kg` gives `9.001kg`, then `9001g`.

### Inline eligibility

A chip is **inline-eligible** only if all of these hold:

- It is a primary: the main result, a trig mode primary, or the remainder of a typed partial result.
- The line class is *inline*.
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

- **Tab** accepts the highlighted chip, but only when it is *acceptable*:
  - A ghost is visible, or
  - The user has explicitly moved the highlight with the arrows.
  - Otherwise Tab inserts a tab and Enter inserts a newline.
  - This keeps phone numbers and dates from hijacking Enter.
- **Arrow keys switch chips** while chips exist:
  - Left goes to the previous chip, and right goes to the next.
  - Up and down keys fall through to normal caret movement.
  - At either end of the list, the key falls through to normal caret movement, which dismisses the ghost.
  - Right arrow does not accept the ghost.
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
| `- 5+5=` | ghost `10` |
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
| `3000mAh / 500 mA =` | `6h` |
| `1 farad * 1 ohm =` | `1s`, obtained by adding the base exponents; no pair lookup |
| `3 gigafarads / 1 nanovolt / 10^18` | `3 m^-4 kg^-2 s^7 A^3`; valid unfamiliar dimensions never fail |
| `760 torr to atm` | approximately `1atm` |

---

## 12. Remaining ambiguities and additional behavior

A digit run glued to the function name is a power when an argument follows (`sin2(30)`, `sin2 x`, `sin2 pi`). A bare `sin2` with nothing after it means sin(2).

When a token pattern like `Ne-N` is genuinely ambiguous, show two chips, scientific notation first, each with its equation rendered as parsed (`1 + 10⁻² = 1.01` and `1 + 1·e − 2 = 1.72`). Everything else shows a single equation in canonical spacing.

Approximate values always keep ≥ 1 decimal and the integer part is never altered. So `pi` → `3.1`, `100/3` → `33.3`, and `9.96…` → `9.96`.

Enter only accepts when a ghost is visible or you've explicitly moved the highlight. Otherwise it's a newline, so a phone number line never eats Enter.

`←`/`↑` and `→`/`↓` switch chips, falling through to the caret at the ends of the list, and `→` no longer accepts.

Lowercase prefix handling:

- One possible reading is accepted and inlined.
- Several readings (`mw` → mW or MW) make it chip-only, one chip per reading.
- Exact-case matches always win and are never treated as ambiguous.

When the user types an explicit `=`, the no-inline rules lift, so `555-1234=` ghosts `-679` and `- 5+5 =` ghosts `10`. Typing `=` signals intent. A lone number like `1=` still shows nothing.

Other requirements:

- Bullet markers are `- * + • –`.
- Infix `nPr`/`nCr` bind tighter than `* /`.
- Modulo requires dimensionless or equal-dimension operands.
- "Exact" means terminating with ≤ 20 significant digits.
- Scientific notation applies to exact values too (≥ 1e15 or < 1e-6).
- One exception: a lone `50%` should not fire.
- Don't autocomplete functions or units in the examples (autocomplete shown in quotes):
  - si"n("
  - ar"ccos("
  - 3 cel"cius"
- OK to autocomplete (inline autocomplete shown in quotes, cells shown in single quotes):
  - `3 celsius to f"ahrenheit = 37.4 fahrenheit`
  - `3 celcius to` suggestions `fahrenheit`, `kelvin`
  - `1kg to` suggestions `lbs = 2.2lbs`, `g = 1000g`, `oz = 35.3oz`
  - `pi = 3.1"4"` then `41` and further digits
  - `pi = 3.14"15"` then `159` and further digits
- Autocomplete constants like pi and e with more precision (pi as 3.14159, e as 2.71828).
- Allow for more precision (for example, keep giving inline autocompletion after `pi = 3.1415927` or after `sin(pi/4)=0.7071`).
- Save the radian/degrees suggestion: if the user last picked radians, inline-suggest radians; if the user last picked degrees, inline-suggest degrees.
- Ensure conversions among torr, atm, Pa, psi, mmHg, and inHg are covered by the unit database and coefficient-based conversion engine.
- Support inline evaluation for `2c to k =`.
