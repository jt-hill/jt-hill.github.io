---
title: "Decimal is sometimes wrong"
date: 2025-11-28
---

[Credkit](https://github.com/jt-hill/credkit) is dropping its usage of the
Python Decimal class. I thought it was the solution to all my problems, but it
was not.

## Decimal is always right

Every financial engineer should learn early: *never use floating point for
money*. The classic example gets trotted out in every intro CS course:

```python
>>> 0.1 + 0.2
0.30000000000000004
```

Whoops! You can imagine how that's a problem. There are cautionary tales about
trading systems accumulating factional penny rounding errors into significant
discrepancies.

If you are working on a Python codebase, you may be pointed to the `Decimal`
class to [solve this problem](https://docs.python.org/3/library/decimal.html).

```python
>>> from decimal import Decimal
>>> Decimal("0.1") + Decimal("0.2")
Decimal('0.3')
```

So when I started building credkit, a Python library for credit
modeling, I did the Responsible Thing and used Python's `Decimal` type
throughout. Exact arithmetic. No rounding errors. A Serious tool for Serious
Professionals.

That was a mistake.

## The Ecosystem Problem

Here's the thing about the `Decimal` Python object: [it's a *Python*
object](https://github.com/python/cpython/blob/main/Lib/_pydecimal.py).

This sounds tautological of course, but the entire Python scientific computing
ecosystem (NumPy, SciPy, pandas, Polars, PyArrow) exists precisely by
*escaping* Python objects. These libraries are thin Python wrappers around C,
C++, or Rust code operating on contiguous arrays of IEEE 754 floats. That's the
whole crux - it's where the speed comes from.

A `Decimal` cannot participate in this, because it is implemented as a Python
object. There is no underlying C object to drop in to. When you put
Decimals in a NumPy array or pandas DataFrame, you get `dtype=object`, which is
just a column of arbitrary Python pointers. No vectorization. No SIMD. Just a
for-loop with all the overhead that implies.

But it's not just performance. Many libraries simply *refuse* to work with Decimal:

```python
# scipy.optimize expects float64
from scipy.optimize import newton
from decimal import Decimal

cfs = [Decimal("-100000"), Decimal("30000"), Decimal("35000")]
newton(lambda r: sum(cf/(1+Decimal(str(r)))**i for i,cf in enumerate(cfs)), Decimal("0.1"))
# TypeError: unsupported operand type(s) for *: 'decimal.Decimal' and 'float'
```

And here's where it can get insidious: some libraries *appear* to support
Decimal but don't really. PyXIRR's type hints show `Amount = Union[int, float,
Decimal]`, which looks like full Decimal support. But watch what happens:

```python
from pyxirr import xirr
from decimal import Decimal
from datetime import date

dates = [date(2020, 1, 1), date(2021, 1, 1), date(2022, 1, 1)]
amounts = [Decimal("-1000.00"), Decimal("500.00"), Decimal("600.00")]

result = xirr(dates, amounts)
print(type(amounts[0]))  # <class 'decimal.Decimal'>
print(type(result))      # <class 'float'>
```

NumPy Financial and PyXIRR both accept your Decimals, convert them to C/Rust
floats at the boundary, do all the math in float64, and hand you back a float.
Your precision was silently discarded. The API gives you the illusion of exact
arithmetic while quietly not providing it.

But even if the library supports it perfectly, *you* might not.

```python
>>> from decimal import Decimal
>>> Decimal("100.10")
Decimal('100.10')
>>> Decimal(100.10)
Decimal('100.0999999999999943156581139191985130310058593750')
```

Miss the quotes and you've embedded the float representation error into your
"exact" Decimal. The whole point of using Decimal was to avoid this, and now
you have to be paranoid about string quoting every single numeric literal. Sure
hope you never forget!

I was paying the cost of `Decimal` everywhere (the slow arithmetic, the clunky
constructors, the constant `Decimal("300000.00")` instead of just `300000.0`)
while losing the precision at every boundary where I actually needed to *do*
something with the numbers.

## Do I really need it?

At this point you might reasonably ask: okay, but does float64 actually work
for financial calculations? Don't you get horrifying rounding errors
compounding across 30-year mortgages?

So I tested it.

**Single Loan (30-year mortgage, 360 payments):**

```markdown
Final balance (Decimal): $-0.0000000000
Final balance (Float64): $-0.0000000465
Difference: $0.000000046 (46 nano-cents)
Match when rounded to cents: True
```

After 360 compound calculations, float64 has accumulated an error of 46
billionths of a cent. When you round to pennies (which you have to do anyway,
because that's what money actually is) they're identical.

**Portfolio interest calculation (1000 loans × 360 payments each):**

```markdown
Decimal total: $348,905,505.58
Float64 total: $348,905,505.58
Difference: $0.00
```

**Daily interest accrual (30 years = 10,950 compounding periods):**

```markdown
$300,000 at 6% compounded daily for 30 years
Decimal final: $1,814,625.78
Float64 final: $1,814,625.78
Difference: $0.00
```

**Accumulation of small amounts (sum a million pennies):**

```markdown
Sum of 1,000,000 × $0.01
Decimal: $10,000.00
Float64: $10,000.00
Difference: $0.00
```

Even after 360,000 interest calculations across a portfolio, nearly 11,000
daily compounding operations, or a million additions of $0.01, the results
match to the penny.

## The Performance Reality

The precision difference is unmeasurable. The performance difference is not.

```markdown
1000 loan amortizations (360 payments each):
  Decimal: 110ms
  Float64: 15ms
  Speedup: 7x

Pandas aggregation (100k rows):
  Decimal (object dtype): 891ms
  Float64 (native dtype): 7ms
  Speedup: 127x
```

For interactive analysis in Jupyter notebooks and dashboards, this is the
difference between "responsive" and "go get coffee."

## What Industry Actually Does

Once I started looking, I noticed that basically *everyone* doing financial
analytics uses float64:

- **QuantLib**: The standard open-source quant finance library defines `Real` as
`double` in its source.
- **R**: R has no Decimal type at all. The language stores all real numbers as
double-precision floats. And R has long been the workhorse for analytical and
computational modeling problems
- **Every pandas/NumPy financial analysis ever**: float64 by default.

The places that use fixed-point decimal arithmetic are accounting systems:
general ledgers, transaction processing, regulatory reporting where everything
must reconcile exactly. But credkit isn't an accounting system. It's an
analytics library. NPV calculations, cash flow modeling, IRR, portfolio
metrics. The output is "this loan is worth approximately $287,342" not "account
47291 has a balance of $287,342.18 that must match the ledger."

I was demanding accounting precision for analytics work, and making everything
more painful for no real benefit.

## The Lesson

Best Practices do not absolve you from needing to actually understand the scope
and bounds of your problem, and "never use floats for money" is a heuristic,
not a divine commandment. It's advice for transaction processing where
everything must reconcile exactly.

But credkit is not a ledger.

After testing, the actual error I found was 46 nano-cents on a single
loan. Across a portfolio of 1000 mortgages with 360,000 interest calculations,
the difference was still zero cents.

Is your loss forecast accurate to the nano-cent? Do you know your prepayment
rate down to the milli-basis-point? If not, you're measuring ingredients to the
microgram while guessing at the oven temperature.

The number type doesn't matter. [Getting the model right
does](mailto:jth@jt-hill.com).

---

*[The benchmark code is
available](https://gist.github.com/jt-hill/839e1a6c51d28cbcb8fc69e6143193eb).
Credkit is on [PyPI](https://pypi.org/project/credkit/).*
