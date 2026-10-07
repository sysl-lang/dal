# dal

A dynamic numeric tower for sysl: a `Number` that is a machine integer until it overflows, a big
integer after that, an exact fraction where a division does not come out even, and a binary float
where a program asks for one -- with the rules that move a value between those kinds chosen by a
`Policy`, because two languages built on the same tower rarely agree on what `7 / 2` is.

```
dependencies {
  dal { git = "github.com/sysl-lang/dal", version = "0.1.1" }
}
```

The coordinate names the package, `dal`; the module a program imports is `sh.sysl.dal`. It needs
sysl `0.1.0-alpha.2` or later and depends on nothing but the standard library: the arithmetic
underneath is `sysl.math.bigint`, `sysl.math.rational` and `sysl.math`'s checked integer operations.
What this package adds is the *dispatch* -- which kind an operation runs in, what happens at its
edges, and what kind the answer comes back as.

```
import sh.sysl.dal.*

val tower = funl()

tower.div(Int(7), Int(2))                       // Ok(Rat(7/2))
tower.add(Int(9223372036854775807), Int(1))     // Ok(Big(9223372036854775808))
tower.mul(Rat(ratio(2, 3)), Int(3))             // Ok(Int(2)) -- demoted
tower.div(Int(1), Int(0))                       // Err(DivisionByZero)
iso_prolog(false).div(Int(7), Int(2))           // Ok(Real(3.5))
compare(Rat(ratio(1, 3)), Real(0.3333333333333333))  // Some(1): the double is below a third
```

## The kinds

| variant | holds | exact? |
|---|---|---|
| `Int(n: long)` | a 64-bit integer -- the fast path, one overflow check per operation | yes |
| `Big(n: BigInt)` | an integer of any size | yes |
| `Rat(q: Rational)` | a fraction in lowest terms over two `BigInt`s, denominator positive | yes |
| `Real(x: real)` | an IEEE 754 double | no |

`Number` is an ordinary enum: a caller matches on it and builds one with the variant. A value a
caller builds is taken as it is; every value an operation *returns* has been put through the policy
(below), so a program that only ever builds `Int`s and `Real`s and lets the tower make the rest only
ever sees canonical values.

## The lattice

```
Int  ──overflow──▶  Big  ──inexact /──▶  Rat  ──meets a Real──▶  Real
 ▲                   │                    │
 └──fits i64─────────┘                    │
 ▲                                        │
 └───────────── denominator 1 ────────────┘
```

**An operation on two numbers runs in the higher of their two kinds**: `Int` with `Big` is done over
`BigInt`, either with a `Rat` over `Rational`, either with a `Real` over doubles. Then:

- **An `Int` operation that overflows is redone in `Big`.** The check is the checked arithmetic in
  `sysl.math`, so the common case costs one branch. `long.min / -1`, `-long.min` and
  `abs(long.min)` are the three single-operand overflows and all three come back as a `Big`.
- **An exact result is demoted to the smallest exact kind that holds it** (when `demote` is on): a
  `Big` that fits 64 bits becomes an `Int`, a `Rat` whose denominator is 1 becomes an integer.
- **An inexact result is never demoted.** `Real(2.5) * Int(2)` is `Real(5.0)`, not `Int(5)`.
- **An exact value meets a `Real` by becoming the nearest double** -- correctly rounded, through
  `rational.to_real`, never through a double computed from the parts.

## Operations

All arithmetic is a method on the `Policy`, because the policy is what decides the answer:

| method | meaning |
|---|---|
| `add`, `sub`, `mul` | the exact result, in the higher kind |
| `div` | `/` -- what it does with two integers is the policy's `division` (below) |
| `quot`, `rem` | truncating division and its remainder: `-7 quot 2 = -3`, `-7 rem 2 = -1` (sign of the dividend) |
| `floor_div`, `modulo` | floor division and its remainder: `-7 floor_div 2 = -4`, `-7 modulo 2 = 1` (sign of the divisor) |
| `neg`, `abs` | negation and magnitude |
| `pow` | integer power: the exponent must be an integer of any exact kind |

Each answers `Result[Number, NumberError]`. **Nothing panics**: a zero divisor, an overflow the
policy refuses and a kind the policy leaves out are all an `Err`.

**`quot`, `rem`, `floor_div` and `modulo` on non-integers** (when `integral` is off): over
rationals the quotient is the exact integer `trunc(a/b)` or `floor(a/b)` and the remainder is
`a - b*q`, exactly; over doubles the remainder is C's `fmod` (exact) adjusted to the divisor's sign
for `modulo`, and the quotient is the double holding `(a - r) / b`.

**`pow`** takes an exponent that is an integer of any exact kind (a `Rat` with denominator 1
counts). A negative exponent gives the reciprocal, which is a `Rat` unless the base is `±1`. A `Real`
base with an exact exponent is C's `pow` -- the exponent is exempt from the `mixing` rule, since an
integer exponent is not a second operand of a different kind but a count. Refusals:
`0 ^ -n` is `DivisionByZero`; a `Real` exponent or a fractional one is `NotAnInteger`; a result
that would need more than 2^26 bits (eight megabytes of digits) is `ExponentTooLarge` rather than an
allocation that takes the program down.

**Division by zero is an error value for every kind**: always in a division that runs exactly, and
in one that runs in doubles (either operand a `Real`) unless the policy asks for IEEE results
(`ieee`), in which case `1.0 / 0.0` and `1 / 0.0` are `Real(inf)` while `1 / 0` is still refused. The integer-division family (`quot`, `rem`, `floor_div`, `modulo`) refuses a zero
divisor even under `ieee`: there is no integer for `1 quot 0` to round to.

### Comparison and equality

Free functions, because no policy changes what is larger:

| function | answers |
|---|---|
| `compare(a, b) -> Option[int]` | `Some(-1)`, `Some(0)`, `Some(1)`; `None` when either is a NaN |
| `equal`, `less`, `less_eq`, `greater`, `greater_eq` | the obvious `bool`s; every one is `false` with a NaN |
| `a == b` (`impl Eq`) | **identity**: same kind and same value |

**Comparison across kinds is exact.** An exact value is compared with a double at the double's
exact binary value -- `1/3` is *above* `0.3333333333333333`, and `9007199254740993` is above
`9007199254740992.0` -- never by rounding the exact side to a double, which would call two different
numbers equal. An infinity is above or below every exact value.

**`equal` is numeric, `==` is not.** `equal(Int(1), Real(1.0))` is `true`, as FunL's `==` and
Prolog's `=:=` want; `Int(1) == Real(1.0)` is `false`, as unification and Prolog's `==` want.
`==` treats a NaN as identical to a NaN (one value is identical to itself), and `0.0` and `-0.0` as
identical (they are `==` as doubles). `equal` follows IEEE: a NaN equals nothing.

**There is no `impl Hash`.** Whether `1` and `1.0` hash alike is the same question as which equality
a table keys on, and that is the consumer's.

### Text

| function | does |
|---|---|
| `policy.parse(s) -> Result[Number, NumberError]` | reads a number |
| `to_string(n)` and `impl Display` | writes one |

`parse` reads an optional sign and then one of: digits (`Int`, or `Big` past 64 bits); digits `/`
digits (a `Rat`, put through the policy, so `"4/2"` is `Int(2)`); digits with a fraction or an
exponent (`Real`); or `inf` / `nan`. Nothing else -- no spaces, underscores, radix prefixes or a bare
`.5` -- is `Syntax`. A kind the policy leaves out is refused by the same errors as arithmetic, and
`"1/0"` is `DivisionByZero`.

`to_string` writes an `Int` and a `Big` in decimal, a `Rat` as `n/d`, and a `Real` as **the
shortest decimal that reads back as the same double** (`sysl.display_real_shortest`), with a `.0`
added where the digits alone would read back as an integer: `1.0`, `0.1`, `1.0e+23`, `-0.0`,
`inf`, `-inf`, `nan`. So `policy.parse(to_string(n))` is `n` again for every `n` the policy admits.

## The policy

```
struct Policy
    bigints: bool
    rationals: bool
    floats: bool
    division: Division
    demote: bool
    mixing: bool
    ieee: bool
    integral: bool

policy(bigints = true, rationals = true, floats = true, division = Exact,
       demote = true, mixing = true, ieee = false, integral = false) -> Policy
```

`policy()` is the default tower, and a variation names only what it changes:
`policy(division = ToFloat, ieee = true)`. A `Policy` is a plain value with public fields.

| field | when on / what it picks | default |
|---|---|---|
| `bigints` | an integer past 64 bits becomes a `Big`; off, it is `Overflow` | on |
| `rationals` | a fraction made out of integers -- an inexact exact quotient, a negative power -- becomes a `Rat`; off, it is `NeedsRational`. A `Rat` *operand* is never refused: see below | on |
| `floats` | `Real`s exist; off, any operation that would produce one is `NeedsFloat` | on |
| `division` | what `/` gives for two integers -- see below | `Exact` |
| `demote` | an exact result is demoted to the smallest exact kind that holds it; off, a `Big` stays a `Big` and a `Rat` a `Rat` (`Rat(1/1)`) -- though an operation on two `Int`s, an integer literal and a power of an `Int` still answer an `Int` wherever it fits, since nothing left a kind | on |
| `mixing` | an exact operand meets a `Real` by becoming a double; off, it is `MixedKinds` | on |
| `ieee` | a non-finite double is a result like any other; off, `inf` is `FloatOverflow`, `nan` is `Undefined`, and a `Real` zero divisor is `DivisionByZero` | off |
| `integral` | `quot`, `rem`, `floor_div` and `modulo` refuse a non-integer with `NotAnInteger` | off |

**`Division`** is what `/` does with two integers; a
`Rat` or a `Real` operand makes `/` an exact or a float division whatever it says:

| `Division` | `7 / 2` | `6 / 2` | used by |
|---|---|---|---|
| `Exact` | `Rat(7/2)` | `Int(3)` | FunL, Scheme, SWI-Prolog with `prefer_rationals` |
| `IntOrFloat` | `Real(3.5)` | `Int(3)` | ISO Prolog |
| `ToFloat` | `Real(3.5)` | `Real(3.0)` | Python 3, JavaScript |
| `Truncate` | `Int(3)` | `Int(3)` | C, Icon |
| `Floor` | `Int(3)` | `Int(3)` | (`-7 / 2` is `-4`) |

**The float `IntOrFloat` and `ToFloat` produce is the correctly rounded quotient**, computed as an
exact fraction and rounded once, not the quotient of two rounded doubles. `2^55 + 3` is not a
double -- as one it is `2^55` -- so `(2^55 + 3) / 3` done in doubles is `12009599006321322.0`; the
true quotient `12009599006321323.67` is nearer `12009599006321324.0`, which is what this answers.

**Where two fields contradict, the narrower one wins at the point of use**: `division = Exact` with
`rationals` off makes `7 / 2` a `NeedsRational` while `6 / 2` is still `Int(3)`. Nothing is checked
when a policy is built, so a policy is a plain value a caller may write field by field.

**`rationals` off means no fraction is *made*, not that a rational is refused.** A `Rat` that reaches
an operation -- one a host language made under another policy and handed over -- is a number like
any other, and arithmetic on it stays exact: `1/2 + 1` is `Rat(3/2)` and `1/2 / 2` is `Rat(1/4)`,
while `1 / 2` still follows `division`. This is SWI-Prolog's rule (`1r2 + 1` is `3r2` with
`prefer_rationals` off). `parse` and `of_rational` still refuse a fraction, since each makes one.

### What FunL chooses -- `funl()`

The defaults, which is not an accident: FunL's design is the reason the defaults are what they are.
**Exact until a program asks for inexactness** -- integers grow without bound, `7 / 2` is `7/2`, an
exact result demotes, a `Real` never does, and `1 == 1.0` is numeric (`equal`) while unification
is not (`==`). Non-finite doubles are errors, and `%` works on reals (`integral` off).

### What ISO Prolog chooses -- `iso_prolog(prefer_rationals)`

| field | `iso_prolog(false)` | `iso_prolog(true)` |
|---|---|---|
| `bigints` | on (unbounded integers, as SWI; ISO permits a bounded one) | on |
| `rationals` | off | on |
| `floats` | on | on |
| `division` | `IntOrFloat`: `7 / 2` is `3.5`, `6 / 2` is `3` | `Exact`: `7 / 2` is `7r2` |
| `demote` | on | on |
| `mixing` | on: `1 + 2.0` is `3.0` | on |
| `ieee` | off: `evaluation_error(float_overflow)`, `(undefined)`, `(zero_divisor)` | off |
| `integral` | **on**: `//`, `mod`, `rem` and `div` are `type_error(integer, X)` on a float | on |

`prefer_rationals` is SWI-Prolog's flag of the same name. With it off, a negative integer power of
an integer other than `±1` is `NeedsRational` -- ISO's `type_error` for `2 ^ -1` -- and a rational
operand still computes exactly, as in SWI. Prolog's `//` is
`quot` and its `div` is `floor_div`.

**Mapping errors to ISO's**: `DivisionByZero` → `evaluation_error(zero_divisor)`, `FloatOverflow` →
`evaluation_error(float_overflow)`, `Undefined` → `evaluation_error(undefined)`, `Overflow` →
`evaluation_error(int_overflow)`, `NotAnInteger` → `type_error(integer, X)`.

### Another language -- an illustration, not a preset

A JavaScript-shaped tower is `policy(bigints = false, rationals = false, division = ToFloat,
ieee = true)`: every division is a double, `1 / 0` is still `DivisionByZero` (an exact division)
while `1.0 / 0.0` is `inf`. slate may pick a
subset like this one; it is not shipped as a preset until a consumer picks it.

## Errors

```
enum NumberError
    DivisionByZero    // a zero divisor
    Overflow          // an integer past 64 bits, with bigints off
    FloatOverflow     // an infinite double, with ieee off
    Undefined         // a NaN, with ieee off
    NeedsRational     // a fraction made out of integers, with rationals off
    NeedsFloat        // a double, with floats off
    MixedKinds        // an exact operand meeting a Real, with mixing off
    NotAnInteger      // integer division of a non-integer with integral on, or a fractional exponent
    ExponentTooLarge  // a power whose result would pass 2^26 bits
    Syntax            // parse: not a number
```

Every one has a `describe()` and `impl Display`.

## Conversions

| function | answers |
|---|---|
| `policy.of_big(n)` | a `BigInt` as a `Number`, through the policy (so a small one is an `Int`) |
| `policy.of_rational(q)` | a `Rational` likewise |
| `to_real(n) -> real` | the nearest double, correctly rounded |
| `to_long(n) -> Option[long]` | the integer, where it is one that fits 64 bits |
| `to_rational(n) -> Option[Rational]` | the exact value, for every exact kind; `None` for a `Real` |
| `is_integer(n)`, `is_exact(n)`, `sign(n)` | the predicates the operations themselves use |

## What is not here, and why

- **Decimal** (`sysl.math.decimal`) -- FunL's design has a `Dec` kind above `Real`. It needs a
  precision carried in the policy and its own rounding rules at every boundary, and is the next
  release's work rather than a bolt-on to this one.
- **Complex numbers** -- the old Scala DAL has them; nothing that consumes this package does yet.
- **Bitwise operations, gcd, square roots, transcendental functions** -- the integer ones are a
  call to `sysl.math.bigint` away once a value is known to be an integer, and the float ones a call
  to `real`'s own methods; a tower adds nothing to them but dispatch, which a consumer can write in
  a line once it has `to_long`, `to_rational` and `to_real`.
- **Hashing** -- see Comparison above.

## Tests

`sysl test .` runs every operation over every pair of kinds, the 64-bit edges (`long.max + 1`,
`long.min - 1`, `long.min / -1`, `-long.min`, `long.min rem -1`), division by zero in every kind and
every operation that divides, both presets and every policy field on and off, the round trip through
text, and every error path. Every expected value is worked by hand or is a property of the
definition (`a = b*quot + rem`, `0 <= modulo < b`); none was produced by this package.

## License

ISC
