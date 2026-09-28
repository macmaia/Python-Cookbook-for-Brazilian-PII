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
    description: "The limits of check-digit detection: names, addresses and clinical facts are not detected, and absence of a finding is not evidence of absence."
---

# 7. What none of this does

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

This chapter exists because a book that only shows what works is not usable at work. What follows is not a
footnote of caveats, it is the outline of the tool, and knowing the outline is what separates using it from
trusting it.

## It does not find names, addresses or clinical facts

```{code-cell}
import tarja

text = ("Maria da Silva, at Av. Presidente Juscelino Kubitschek 1909, suite 181, "
        "diagnosed with hypertension at a consultation on 12/03/2024.")
print(tarja.find(text))
```

Empty. A name, an address, a date and a health condition are all there, and none is detected.

This is not a gap to be filled later, it is a different problem. A structured identifier has a formation rule,
which is why arithmetic can recognise it. A name does not: "Silva" is a surname and also appears in street
and municipality names. An address does not. Recognising those needs a language model trained for the task, with its own
error rate that has to be measured separately and that is considerably worse than a check digit's.

Anyone who needs both combines two tools, and measures both.

## Absence of a finding is not evidence of absence

This is the most important sentence in the book, and it holds even with the tool working perfectly.

The report says "no identifiers found". That means the detector did not find any. It does not mean there are
none. There may be an identifier of a type it does not know, in a format the pattern does not cover, broken by
a line wrap, or inside a PDF that is really an image.

If anyone is going to use your report to assert that a corpus is clean, that sentence needs to be written in
the report, in plain words, before somebody concludes the opposite.

## A valid digit does not mean the person exists

```{code-cell}
print(tarja.validate("BR_CPF", "529.982.247-25"))
```

That says the number is well formed. It does not say it was issued, or to whom. A randomly generated value that
closes the arithmetic passes exactly the same way, which is how the examples in this book are produced.

The converse holds too: nothing here queries an external register, nothing resolves an identifier to a person.
That boundary is deliberate. A tool that confirmed "this number belongs to so-and-so" would be more useful to
whoever wants to protect people and immediately more useful still to whoever wants to harvest them.

## Scanning noise breaks detection

The most uncomfortable test is characters swapped by optical character recognition, capital O for zero and
lowercase l for one.

In tarja's published measurement, the subset carrying that kind of noise scores an **F1 of 0.266**, against
figures above 0.95 on clean text. That is not degradation, it is failure.

The reason is that the normaliser handles Unicode look-alikes, accents and width, and **not** the letter/digit
confusion, because doing that without context would create false positives everywhere: not every O among
digits is a zero.

Practical consequence: if your corpus contains scanned documents, extract the text with a good tool and treat
its output as suspect. Do not carry the clean-text accuracy over to it.

## The vendor's measurement is not yours

Every tool publishes an accuracy figure. It was obtained on a test set that is not your text.

Chapter 3 showed this concretely: a score threshold separates good hits from bad **when the false positive is
far from any context word**, which is a property of the text and not of the method. In a corpus of labelled
forms it works well, in running prose it works worse, and documentation has no way to tell you which one you
have.

That is why chapter 6 exists. Measuring on your own text is not excessive diligence, it is the only way to
know.

## None of this settles your legal obligations

Finding and masking personal data is a technical measure. It remains your responsibility to have a legal basis
for the processing, to keep records of processing operations, to set and honour retention periods, to answer a
data subject asking for access or deletion, and to notify an incident where required.

And as chapter 4 showed, replacing values with stable labels is pseudonymisation: the data is still personal
and still inside the law.

A tool reduces risk. It does not transfer responsibility, and no report from it is a certificate of compliance.

## The vault does not survive an index

The `reveal()` of chapter 4 relies on a vault that is local to the process, single use, and valid for one hour.
That was deliberate: a persistent map from token to value would be a database of personal data, which is exactly
what pseudonymisation exists to avoid.

The consequence is a limit of scope, not a defect. `mask` works in any pipeline, including ingestion into a
vector store. `reveal` only works inside the same process and the same hour. Anyone who needs the original value
back weeks later needs a key custodian of their own, and chapter 8 covers that.

## Dual use

A detector of Brazilian identifiers is the same instrument for two opposite purposes. Whoever wants to protect
uses it to find and mask. Whoever wants to harvest uses it to find and extract.

There is no technical version that serves one and not the other: the capability is identical, only what you do
with the output differs. That is why the library does not resolve identifiers to people, does not query
external registers and does not make bulk export of values convenient, and why hiding the value is the default
rather than showing it.

None of that prevents misuse, and it does not pretend to. It moves the default, and leaves misuse as an
explicit choice by whoever makes it rather than an accident by whoever did not notice.

## Takeaway

It does not find names or addresses. It does not prove absence. It does not confirm that anyone exists. It
fails on noisy scanned text. It does not transfer the vendor's accuracy to your case, nor your legal
responsibility to the tool.

Every limit above applies to one text at a time. Chapter 8 covers the case where the text does not pass through
but stays: ingestion for retrieval, where splitting into chunks destroys the very arithmetic that would have
recognised the data.
