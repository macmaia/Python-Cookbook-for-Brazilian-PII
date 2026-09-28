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

# Python Cookbook for Brazilian PII

Your product has Brazilian customers. Their identifiers are in your support tickets, your logs, your data
warehouse and the prompts your application sends to a model. Your PII tooling, whichever one you chose, was
built for Social Security numbers and NHS numbers, and it does not know what a CPF is.

This book is about closing that gap, in Python, with code that runs.

## Who this is for

Engineers outside Brazil, or on international teams, who handle Brazilian customer data and have been asked to
find it, mask it, or keep it out of somewhere. You do not need to read Portuguese and you do not need to know
Brazilian administrative law. You do need to know that a Brazilian identifier is not a number, it is a number
with a rule.

There is a companion book written for a Brazilian audience, which frames the same material around the LGPD and
the work of cleaning up a system already in production:
[Python para Anonimizar Dados Pessoais](https://macmaia.github.io/Python-para-Anonimizar-Dados-Pessoais/).
This one is not a translation of it.

## Why the usual tools come back empty

Presidio, the cloud data-loss-prevention services and most commercial scanners work the same way: a registry
of recognisers, one per entity type, each bound to a language. There are recognisers for `US_SSN`, `UK_NHS`
and `IN_AADHAAR`. There are none for CPF, CNPJ, the health card or the court case number. Where there is no
recogniser there is no detection, and the tool does not tell you it was not looking.

So the scan comes back clean, and clean is the most expensive wrong answer available.

## What you will be able to do

Explain why a regular expression for eleven digits produces a report nobody can use, with a measurement rather
than an opinion. Implement a Brazilian check digit from scratch, so you can judge an implementation instead of
trusting one. Wire Brazilian entities into Presidio, if that is already your stack. Decide between redacting,
pseudonymising and tokenising, knowing what each one lets you do afterwards. Strip identifiers before text
reaches a language model, and put them back in the answer. And measure your own error rate in both directions.

## Running the code

```bash
pip install tarja
```

From chapter 2 the book uses [tarja](https://macmaia.github.io/tarja/), an Apache 2.0 library with no runtime
dependencies. Chapter 2 writes the CPF rule by hand first, on purpose: you should understand the mechanism
before you decide to trust someone else's implementation of it.

Every chapter has a button to open it in Colab and run it there.

## What this book does not cover

Names, addresses and clinical facts in prose. Those need named entity recognition, which is a different
problem with a different error profile. Chapter 7 is about the limits, and it exists because a book that only
shows what works is not usable at work.

---

By [Maria Alice Maia](https://github.com/macmaia). Text under CC BY 4.0, code under Apache 2.0.
