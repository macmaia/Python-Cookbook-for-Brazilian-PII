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

# 3. When there is no check digit

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

Not every identifier carries a check digit. Postcodes, licence plates and phone numbers are shapes and nothing
more. And some that do carry one still collide with legitimate numbers, because the space of valid values is
large enough for coincidence.

In both cases the only remaining information is what is written around the number.

## The easy case

```{code-cell}
import tarja

print(tarja.find("22290-140"))
```

Nothing. Five digits, a dash and three digits is a shape that a price band, a part number or a range could
have. On its own it is not evidence.

```{code-cell}
for m in tarja.find("CEP 22290-140"):
    print(m.entity, "score", m.score, "context", m.has_context)
```

The word changed everything. Same digits, and what appeared was the context.

That is the difference between **requiring** context and **scoring** with it. The postcode requires it: without
the word it is not reported at all. The licence plate does not, it appears with a lower score and rises when
context is present.

```{code-cell}
for text in ["ABC1D23", "vehicle with plate ABC1D23"]:
    hits = tarja.find(text)
    print(f"{text!r:32} -> score {hits[0].score if hits else None}")
```

## How the comparison works

The context window looks at some characters before and after the match and searches there for the entity's
words. The comparison runs over lowercase, accent-free text, which is why all of these work:

```{code-cell}
for text in ["CEP 22290-140", "cep 22290-140", "Cep: 22290-140", "CÉP 22290-140"]:
    print(f"{text!r:22} -> {len(tarja.find(text))} hit(s)")
```

This matters more than it looks. Real text has inconsistent case, wrong accents, abbreviations and punctuation
run together. A literal comparison would miss most of it, and whoever measured would conclude the detection is
poor when it was the preprocessing.

## The hard case, and it is an honest one

Now the example this chapter is named after.

A voter registration number has twelve digits and a check digit. Watch what happens to a run of years:

```{code-cell}
for m in tarja.find("Reference years: 1999 2001 2003 2005"):
    print(m.entity, "score", m.score, "context", m.has_context)
```

`1999 2001 2003` **passes the check digit**. That is not a bug in the detector or in the algorithm: the space
of valid registrations is large, and four consecutive years fall inside it by coincidence.

The check digit solved chapter 1's problem and does not solve this one. Hence context.

```{code-cell}
registration = "7097 1071 1422"   # synthetic, valid check digit

for text in [registration, f"Inscricao: {registration}", f"titulo de eleitor {registration}"]:
    m = tarja.find(text)[0]
    print(f"{text!r:42} score {m.score}  context {m.has_context}")
```

With context, 0.95. Without, 0.8. So a score threshold should separate them, and that is what almost everyone
does:

```{code-cell}
text = f"Eleitora, inscricao {registration}. Anos de referencia: 1999 2001 2003 2005."

for m in tarja.find(text, min_score=0.9):
    print(f"score {m.score} context {m.has_context} -> {text[m.start:m.end]!r}")
```

**Both survive.** The run of years also scored 0.95, because the context word sits within sixty characters of
it too. The window measures proximity, not grammatical relation, and has no way to know that the word refers
to the other number.

Move them apart and the behaviour changes:

```{code-cell}
far = (f"Eleitora, inscricao {registration}. "
       + "Filler sentence with nothing of interest. " * 3
       + "Anos de referencia: 1999 2001 2003 2005.")

for m in tarja.find(far, min_score=0.0):
    print(f"score {m.score} context {m.has_context} -> {far[m.start:m.end]!r}")
```

Now it works: 0.95 for the registration, 0.8 for the years.

## What to take from this

A score threshold separates true from false **when the false positive is far from any context word**. That is a
property of your text, not of the method.

In a corpus where every identifier sits in a labelled field with the label next to it, separation works very
well. In running prose, where the word could be anywhere in the paragraph, it works worse. You do not find out
which one you have by reading documentation. You find out by measuring, and chapter 6 shows how.

This is the kind of limitation a vendor does not volunteer. They publish accuracy on a test set and let you
assume it transfers to your text. It does not, and assuming it does is how most personal-data detection
projects end with a report nobody trusts.

## Choosing the window is choosing an error

A wider window catches the label that sits far away, so it reduces false negatives. A wider window also reaches
words unrelated to the number, so it increases false positives. There is no correct value, only which of the
two errors costs you more.

## Takeaway

Without a check digit, context is all there is, and it measures proximity rather than meaning. A high score
means "there is a context word nearby", and nearby does not mean related.

Next: you found the identifier. What goes in its place.
