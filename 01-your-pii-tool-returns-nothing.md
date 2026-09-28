---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
kernelspec:
  display_name: Python 3
  language: python
  name: python3
myst:
  html_meta:
    description: "Why Presidio and other PII tools return nothing on Brazilian text, and why a format-only regex for CPF produces false positives at scale."
---

# 1. Your PII tool returns nothing, and the obvious fix is worse

```{code-cell}
:tags: [skip-execution]
# On Colab, run this first. It is skipped when the book is built, where tarja is already installed.
%pip install -q tarja
```

A support ticket from a Brazilian customer looks like this:

```{code-cell}
ticket = """
Ticket 20240315001 reopened by the customer.
Customer states the invoice is wrong. Taxpayer number on file: 529.982.247-25.
Internal case reference 18273645902, agent badge 30912847561.
"""
print(ticket)
```

There is one piece of personal data in there. A scanner configured for a European or North American entity set
finds none of it, and reports the ticket as clean. Nothing in the report says "I was not looking for Brazilian
identifiers", because the tool has no way to know that it should have been.

## Why is a CPF regex not enough?

Somebody writes a regular expression. Eleven digits, optionally punctuated.

```{code-cell}
import re

CPF_BY_SHAPE = re.compile(r"\b\d{3}\.?\d{3}\.?\d{3}-?\d{2}\b")

for hit in CPF_BY_SHAPE.finditer(ticket):
    print(hit.group())
```

Four hits. One is a taxpayer number. The other three are a ticket number, an internal case reference and an
employee badge.

This is not a contrived example. Brazilian administrative text is dense with long numeric identifiers, and
eleven digits is an unremarkable length for any of them. The ratio in a real corpus is worse than in this
paragraph, because a support system produces far more ticket numbers than taxpayer numbers.

## How much noise does a format regex accept?

How much of that is noise? Generate random eleven-digit strings and count how many the pattern accepts.

```{code-cell}
import random

rng = random.Random(2026)
strings = ["".join(rng.choice("0123456789") for _ in range(11)) for _ in range(100_000)]
by_shape = sum(1 for s in strings if CPF_BY_SHAPE.fullmatch(s))
print(f"{by_shape} of {len(strings)} accepted by shape, i.e. {100*by_shape/len(strings):.0f}%")
```

One hundred per cent. The expression rejects nothing of the right length, because counting digits is all it
can do.

So the choice as it stands is between a tool that finds nothing and a tool that finds everything. Neither is a
report. The first lets you believe there is no personal data. The second, if you act on it, destroys the
ticket numbers and case references your operation runs on, and the first person to notice will ask you to
loosen the rule, which puts the personal data back.

## What the regex cannot see is the check digit

A CPF is not any eleven digits. The last two are **computed** from the first nine, by a published,
deterministic rule. A number where that arithmetic does not close is not a CPF.

That is the pivot of this book. The information that separates a taxpayer number from a ticket number is not
in the shape, it is in the **formation rule**. And Brazilian identifiers are unusually rich in formation
rules: the taxpayer number for individuals (CPF) and for companies (CNPJ), the national health card, the
court case number, the voter registration, the social security number, the vehicle registration, the driving
licence. Nearly all of them carry a check digit.

You can see the effect before learning the rule. Repeat the count using the check:

```{code-cell}
import tarja

valid = sum(1 for s in strings if tarja.validate("BR_CPF", s))
print(f"{valid} of {len(strings)} satisfy the formation rule, i.e. {100*valid/len(strings):.2f}%")
print(f"noise reduction: {by_shape/max(valid,1):.0f}x")
```

From everything to roughly one in a hundred, with no external lookup, no model and no network call. Just
arithmetic that the Brazilian tax authority published decades ago.

Back to the ticket:

```{code-cell}
for hit in CPF_BY_SHAPE.finditer(ticket):
    value = hit.group()
    print(f"{value:18} {'CPF' if tarja.validate('BR_CPF', value) else 'not a CPF'}")
```

The three operational numbers drop out. The taxpayer number stays.

## What a false positive on a CPF costs

The formation rule cuts false positives. It does **not** cut false negatives. An identifier you never looked
for is still sitting in the text. Worse, a CPF typed with one digit wrong fails the arithmetic and disappears
from the report, while remaining real personal data about a real person. Chapter 6 deals with both, because
they cost different things and most commercial tooling does not separate them.

And a valid check digit does not mean the number belongs to anyone. It means the number is well formed. A
randomly generated value that happens to close the arithmetic passes exactly like a real one. Nothing in this
book queries an external register or resolves an identifier to a person, and chapter 7 explains why that
boundary is deliberate rather than a missing feature.

## Does this work with Presidio?

You do not have to replace it. Presidio's architecture is a registry of recognisers, and a recogniser is a
pattern plus a `validate_result` method that returns `True`, `False` or `None`. Returning `False` makes
Presidio **drop** the result, which is exactly the behaviour the check digit needs.

Chapter 2 writes that recogniser, and shows the shortcut for the other fifteen entities.

## Takeaway

A scanner with no Brazilian recognisers returns clean, which is worse than returning wrong. Replacing it with
a shape-matching regular expression produces roughly a hundred false positives for every real finding. The
missing information is the formation rule, and it is public.

Next chapter: the arithmetic, written from scratch, in fifteen lines.
