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

# 4. Redact, pseudonymise, anonymise

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

You found the identifier. What goes in its place. There are three answers, and they are not degrees of the same
thing. They are different decisions about what you can do afterwards.

The question that picks between them is not "how much privacy do I want". It is: **do I need to know that two
records belong to the same person?** and **do I need to get the original value back?**

## Delete it

```{code-cell}
import tarja

doc = "Ticket from Maria, CPF 529.982.247-25. Complaint about the invoice."
print(tarja.mask(doc))
```

The value is gone. No reversal, no counting of distinct people, no joining with anything. Every taxpayer number
in the corpus becomes the same label.

This is the right choice when the identifier plays no part in what comes next. You want to read the tickets to
understand what people complain about, and the number is not involved in that.

## Number them within the document

```{code-cell}
print(tarja.mask("Ticket 3. Holders 123.456.789-09 and 529.982.247-25.", strategy="pseudonym"))
```

Now you can see there are two different people and which appeared first. The text stays readable and counting
stays possible.

That is what the strategy is for: **one call, several identifiers**. Inside that text, equal labels are the
same value and different labels are different values. It delivers what it promises.

The problem appears when the output is used outside that scope, and that happens easily, because the natural
way to process a corpus is one document per call:

```{code-cell}
a = "Ticket 1. Holder CPF 529.982.247-25."
b = "Ticket 2. Holder CPF 123.456.789-09."   # a different person

print(tarja.mask(a, strategy="pseudonym"))
print(tarja.mask(b, strategy="pseudonym"))
```

**Two different people, the same label.** The counter restarts on each call, and `_1` only ever means "the first
CPF in this document".

That is not a defect and the documentation does not claim otherwise: the label is valid **within** one call.
The risk is that the output does not carry that warning with it. A file of ten thousand tickets masked this way
looks comparable and is not, and an analyst joining on the label will conclude they are the same person.

## Derive it from a key

```{code-cell}
import secrets

# A key is a real secret, not a hand-picked string. In production it comes from a secret manager, never from
# the code. Here it is generated on the spot, so the labels below change on every build of this book, which
# is exactly what a key should do.
KEY = secrets.token_hex(32)

print(tarja.mask(a, strategy="pseudonym_stable", salt=KEY))
print(tarja.mask(b, strategy="pseudonym_stable", salt=KEY))
```

Different labels for different people. And the same value, in any document:

```{code-cell}
doc1 = "Ticket from Maria, CPF 529.982.247-25."
doc2 = "Second ticket, same holder, CPF 529.982.247-25."

print(tarja.mask(doc1, strategy="pseudonym_stable", salt=KEY))
print(tarja.mask(doc2, strategy="pseudonym_stable", salt=KEY))
```

Same label. That is what lets you build a dataset about a person without the person's number in it.

The label is not random, it is derived from the value and a secret. Change the secret and everything changes:

```{code-cell}
print(tarja.mask(doc1, strategy="pseudonym_stable", salt=KEY))
print(tarja.mask(doc1, strategy="pseudonym_stable", salt=secrets.token_hex(32)))
```

So **stability holds under one key, and one only**. A document masked last month stops joining with one masked
today if the key changed in between, and nothing breaks and nothing warns: the data simply stops matching.

The four characters in the middle of the label are the key generation marker. Same key, same marker. It is what
lets you notice that two labels came from different keys, rather than discovering it through a statistic that
does not add up.

## What this is, legally

A stable label is **pseudonymisation, not anonymisation**, and under Brazilian law the difference has teeth.

The LGPD puts anonymised data outside its scope, because it no longer relates to an identifiable person.
Pseudonymised data stays inside, because identification is still possible: whoever holds the key reverses it,
and even without the key a stable label lets you follow one person across the whole corpus, which is
re-identification by linkage.

So replacing a taxpayer number with a stable label reduces risk and does not end your obligations. Legal basis,
retention, security and answering the data subject all still apply to the pseudonymised data.

If your goal is to take the data out of the law's scope, `pseudonym_stable` does not do that, and no reversible
substitution does.

## The key is the data

Whoever holds the key turns any label back into the value, for the entire space of taxpayer numbers, because
that space is small enough to enumerate. The key is not configuration, it is the personal data in another form.

Practical consequences: the key does not live in the code or the repository, it comes from a secret manager.
Access to the key is access to the data, and belongs in your access control. And if the key leaks, the whole
masked corpus leaked with it, retroactively.

## Choosing

| I need to... | Strategy |
|---|---|
| read the text, the number does not matter | `redact` |
| count distinct people inside one document | `pseudonym` |
| join records about the same person across documents | `pseudonym_stable` |
| get the original value back, under control | `Vault`, chapter 5 |

And a rule that saves argument: choose the **least** powerful option that solves your case. Every step up
carries one more obligation.

## Takeaway

The three are not levels of protection, they are different capabilities. `pseudonym` does not join and can
mislead anyone who assumes it does. `pseudonym_stable` joins, and is therefore still personal data, under a key
as sensitive as the data itself.

Next: when you need the value back.
