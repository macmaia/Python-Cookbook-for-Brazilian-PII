---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

# 2. Check digits, written from scratch

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

The previous chapter ended on the claim that the missing information is the formation rule. Here you write it.
Fifteen lines, and writing them changes how you judge anybody else's implementation afterwards.

## The arithmetic

The first nine digits of a CPF are the number. The last two are the check: multiply each digit by a descending
weight, sum, take the remainder modulo 11, turn the remainder into a digit.

```{code-cell}
def cpf_check_digit(base: str) -> str:
    """One CPF check digit, computed from the digits before it."""
    n = len(base) + 1
    total = sum(int(d) * (n - i) for i, d in enumerate(base))
    remainder = (total * 10) % 11
    return "0" if remainder == 10 else str(remainder)

print(cpf_check_digit("529982247"))    # first digit
print(cpf_check_digit("5299822472"))   # second, which includes the first
```

For nine digits `n` is 10 and the weights run from 10 down to 2. For ten digits `n` is 11 and they run from 11
down to 2. Same function, called twice, the second call including the digit the first produced.

The `remainder == 10` branch exists because a remainder modulo 11 can be ten, which does not fit in a digit.
The convention is zero.

## The validator

```{code-cell}
def cpf_is_valid(value: str) -> bool:
    d = "".join(c for c in value if c.isdigit())
    if len(d) != 11:
        return False
    if len(set(d)) == 1:
        return False
    return cpf_check_digit(d[:9]) == d[9] and cpf_check_digit(d[:10]) == d[10]

cases = [
    ("529.982.247-25", True),
    ("529.982.247-24", False),   # one digit changed
    ("52998224725",    True),    # unpunctuated
    ("123.456.789-09", True),
    ("111.111.111-11", False),
    ("000.000.000-00", False),
]
for value, expected in cases:
    got = cpf_is_valid(value)
    print(f"{value:18} {str(got):5} {'ok' if got == expected else 'WRONG'}")
```

## The line that looks redundant

Look again at this one:

```python
if len(set(d)) == 1:
    return False
```

It rejects `111.111.111-11` and the other nine repeated-digit strings. It looks like defensive clutter. It is
not:

```{code-cell}
print("check digit computed for 111111111 :", cpf_check_digit("111111111"))
print("check digit computed for 1111111111:", cpf_check_digit("1111111111"))
```

**The arithmetic closes.** `111.111.111-11` satisfies modulo 11 perfectly, and without that line it passes as a
valid CPF, as do `222.222.222-22` and the rest.

This is not a mathematical curiosity. Repeated digits are the most common filler value in existence: a required
field somebody had to get past, a test row left in a database, an import that filled a blank. A validator
without that rule marks every one of them as a taxpayer number, and you will spend an afternoon working out
why ten thousand identical CPFs are in your report.

That rule is not in the algorithm. It is in the practice, and you only meet it by hitting it.

## The other rules, one line each

The same idea recurs with variations, and this is where reimplementing stops being reasonable.

**CNPJ**, the company taxpayer number, is modulo 11 with different weights over twelve digits. Since a recent
change it accepts letters: the computation uses the character's ASCII value minus 48, so `0` is still 0 and `A`
is 17. Older implementations break silently here, because they cast to integer and either raise or drop the
letter.

**PIS, voter registration, vehicle registration and driving licence** are modulo 11 with their own weights and
their own conventions for the remainder. The voter registration has a special case for two of the federal
units.

**The court case number** is not modulo 11 at all. It uses ISO 7064, modulo 97-10, which is a different family.

**The national health card** has its own routine published by the health ministry, with different paths for
provisional and definitive cards.

**Payment cards** use Luhn, which is modulo 10, plus the issuer prefix. Luhn alone accepts roughly one long
number in ten, so without the prefix you are back to chapter 1.

Seven algorithm families, seventeen identifier types, each with a different normative source to find, read and
verify.

## From here, the library

```{code-cell}
import tarja

print(sorted(tarja.ENTITIES))
```

Seventeen types, each with its normative source cited in the code and the date it was checked.

```{code-cell}
print(tarja.validate("BR_CPF",  "529.982.247-25"))
print(tarja.validate("BR_CNPJ", "11.222.333/0001-81"))
print(tarja.validate("BR_CNPJ", "12.ABC.345/01DE-35"))   # alphanumeric
print(tarja.validate("BR_CNJ",  "0000001-83.2017.8.26.0100"))
print(tarja.validate("BR_CNS",  "898 0000 0004 3208"))
```

And finding them in text rather than validating one value:

```{code-cell}
text = ("Contract with company CNPJ 11.222.333/0001-81, "
        "signed by the partner holding CPF 529.982.247-25, "
        "under case 0000001-83.2017.8.26.0100.")

for m in tarja.find(text):
    print(f"{m.entity:10} {m.start:3}:{m.end:<3} score {m.score}")
```

The output does not print the values. That is deliberate, and chapter 6 explains why.

## Into Presidio, if that is your stack

A Presidio recogniser is a pattern plus a `validate_result` that returns `True`, `False` or `None`. The check
digit maps onto it exactly:

```python
from presidio_analyzer import Pattern, PatternRecognizer

class CpfRecognizer(PatternRecognizer):
    PATTERNS = [Pattern("cpf", r"\b\d{3}\.?\d{3}\.?\d{3}-?\d{2}\b", 0.3)]
    CONTEXT = ["cpf"]

    def __init__(self, supported_language: str = "pt"):
        super().__init__(supported_entity="BR_CPF", patterns=self.PATTERNS,
                         context=self.CONTEXT, supported_language=supported_language)

    def validate_result(self, pattern_text: str):
        return cpf_is_valid(pattern_text)   # False makes Presidio DROP the result
```

Returning `False` is the important part: Presidio discards the result rather than lowering its score, which is
what removes the ticket numbers.

Doing that seventeen times is what `tarja-presidio` is for:

```python
pip install tarja-presidio
```

```python
import tarja_presidio
tarja_presidio.register(registry)   # all seventeen entities at once
```

## What you gained by writing it by hand

You are not going to reimplement seventeen algorithms, and nobody should. What you gained is the ability to
read somebody else's implementation and judge it. Look for the repeated-digit rejection. Look for the
remainder-of-ten branch. Look for whether the company number accepts letters. If all three are there, whoever
wrote it knew the problem. If one is missing, you already know what will break.

That is the only honest way to delegate a rule: understand it first.

## Takeaway

A check digit is public arithmetic, writable in fifteen lines, and it is what separates a taxpayer number from
a ticket number. The rule that is not in the algorithm, the rejection of repeated digits, is the one that costs
the most in practice.

Next: what to do when that arithmetic does not exist.
