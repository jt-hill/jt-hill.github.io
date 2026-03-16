---
title: "You don't hate Python. You hate other people's Python."
date: 2026-03-16
---

I apologize for using the forbidden sentence structure, but it's true. If you
haven't written Python since like 2016, you might not have been paying
attention to the ongoing _engoodening_ of the language. In the last 5-10 years
Python has matured from a scripting language appropriate short-lived glue code
to a powerful, expressive, fully-featured language for Serious software
engineering.

## The dark times

Python 2 was bad. Enough ink has been spilled over this that we don't have to
relitigate the question here (`UnicodeDecodeError: 'ascii' codec can't decode
byte 0xc3`). It had to go for the language to have a future. And so you may
remember the Python 2 to 3 migration as the most painful software update since
Y2K: chaos and destruction, rivers of blood, bodies piled in the streets, etc.

Even more than other languages, Python lives and dies by its ecosystem, so the
library ecosystem was the real chokepoint: NumPy didn't support 3 until 2010,
Django until 2013, countless smaller libraries never. The `2to3` tool handled
syntax but not semantics - every piece of I/O code had to decide "is this text
or bytes?" and no tool could decide that for you. The `six` compatibility
library became load-bearing infrastructure. Python 2.7's EOL extended from 2015
to 2020, killing any urgency. In the end, Python 2 died through attrition -
library authors dropped support and holdouts had no choice. It took a decade.
"We don't want a Python 3 situation" is now the universal shorthand in other
language communities for how not to evolve a language.

## The Type system

The first shred of light appeared in 2015-2016 with Python 3.5 and 3.6. PEP 484
added type hints. They were optional and they left a lot to be desired, but
they established a foundation.

Early type hints required ugly imports for basic things (`List[int]`,
`Optional[str]`, `Union[X, Y]`) and forward references had to be quoted
strings. Each release filed this down. 3.9: `list[int]` natively, no imports.
3.10: `X | Y` instead of `Union`. 3.11: `Self` for return types. 3.12: native
generics on functions and classes (`def first[T](items: list[T]`) -> T:), plus
`@override` and the `type` keyword. 3.14 finally, _finally,_ adds lazy
evaluation to fix the circular dependency issues and we can all stop invoking
`from __future__ import annotations` for things that should Just Work.

## Other people's code

So as I have now deftly convinced you, Python is a good language with good
features, and I'm sure you're eager to use them. This gets us to the last
problem. People. Python's type system is optional and incremental - you don't
_have_ to use it consistently (or at all), and many programmers choose not to.
You can still get away with:

```python
def process(data, flag=False, extra=None, **kwargs):
    result = {}
    for k, v in data.items():
        if flag and extra:
            result[k] = v * extra.get('multiplier', 1)
        else:
            result[k] = v
    return result or None
```

What's `data`? A dict of what to what? What's `flag` actually controlling? Why
does it return None sometimes? What's in `kwargs` and does anything even use
it? No one will stop you from writing \*\*kwargs-in, dict-out, bare-except,
mutable-default-argument garbage with no annotations and no docstrings.

And this is a **good thing.** This is why Python won. Sometimes you just need
to get the job done. You want the tool to get out of your way and simply
manipulate some data, run a process, do an experiment. Not everything is meant
to be long-lived and maintained, but when it is, you have simply to use the
convenient tools at your disposal:

```python
from dataclasses import dataclass

@dataclass
class ScalingConfig:
    multiplier: float = 1.0

def scale_values(
    values: dict[str, float],
    config: ScalingConfig | None = None,
) -> dict[str, float]:
    if config is None:
        return values
    return {k: v * config.multiplier for k, v in values.items()}
```

Pyright catches type errors at edit time. Ruff handles linting and formatting
fast enough to run on save. Pydantic enforces types at runtime at API
boundaries. Dataclasses replace formless dicts. Enums and `Literal` kill
stringly-typed dispatch. `Protocol` gives you Go-style structural interfaces.
`pre-commit` gates all of it in CI. None of this existed in usable form in 2016. The gap between "Python lets you write good code" and "your team actually
writes good code" is closeable now, if you close it.
