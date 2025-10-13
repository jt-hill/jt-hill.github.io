---
title: "Introducing Credkit"
date: 2025-10-12
---
# Introducing credkit: Consumer Loan Modeling Without the Pain

I got tired of fighting with QuantLib to model simple mortgages. So I built something better.

## The Problem

QuantLib is great for complex derivatives. But for simple amortizing consumer loans it's complete overkill. You end up spending more time wrestling with schedule builders and leg constructors than actually modeling the loan.

These are relatively instruments, so they should be similarly simple to model

## The Solution

```python
from credkit import Loan, Money, InterestRate

loan = Loan.mortgage(
    principal=Money.from_float(300000.0),
    annual_rate=InterestRate.from_percent(6.5),
    term_years=30,
)

payment = loan.calculate_payment()  # ~$1,896.20/month
schedule = loan.generate_schedule()  # Done.
```

That's it. No ceremony, no fighting with abstractions designed for exotic swaps.

## What You Get

**Domain-specific primitives:**
- `Money`, `InterestRate`, `Spread` types that actually make sense
- Day count conventions, payment frequencies, business day calendars
- Full amortization support (level payment, level principal, interest-only, bullet)

**Decimal precision everywhere:**
- No floating-point rounding errors
- Financial calculations you can trust

**Type-safe and immutable:**
- Frozen dataclasses with full type hints
- Your IDE will actually help you
- No accidental mutations

**Cash flow modeling:**
- Generate schedules with principal/interest breakdown
- Discount curves for NPV calculations
- Filter and aggregate however you need

## Status (v0.2.0)

148 passing tests. Works end-to-end: create a loan → generate schedule → calculate NPV.

Currently USD-only, focused on US consumer lending (mortgages, auto, personal loans).

## What's Next

- Credit risk metrics (PD, LGD, ratings)
- pandas integration
- Prepayment modeling

## Try It

```bash
pip install credkit
```

GitHub: [github.com/jt-hill/credkit](https://github.com/jt-hill/credkit)  
PyPI: [pypi.org/project/credkit](https://pypi.org/project/credkit/)
License: AGPL-3.0 (commercial options available)

## Feedback?

This is early. If you work in or near credit and have ideas or pain points, let me know. What would make your life easier?

