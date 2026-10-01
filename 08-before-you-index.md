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
    description: "Why redacting before indexing into a RAG store differs from redacting before a single call: chunking splits the CPF, the question also carries PII, and the index persists."
---

# 8. Before you index, not just before you ask

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

Chapter 5 dealt with text on its way to a model in a single call. That is no longer where most of the work is.
Most of it is in ingestion for retrieval, the RAG pipeline: the document is split into chunks, each chunk
becomes a vector, and the index stays.

The difference is not one of degree. Redacting before a call is a decision about text that passes through.
Redacting before indexing is a decision about text that **remains**, somewhere it is hard to remove from.

## Why does chunking break detection?

Because a check digit needs all eleven digits together, and the split does not know that.

```{code-cell}
import tarja

document = "Agreement signed by the holder, CPF 529.982.247-25, on 12/03/2024."
size = 40

for c in [document[i:i + size] for i in range(0, len(document), size)]:
    print(f"{c!r:45} -> {[m.entity for m in tarja.find(c)]}")

print()
print("whole document ->", [m.entity for m in tarja.find(document)])
```

The whole document contains a CPF. Neither chunk does. The cut landed inside the number, and what is left on
each side is not a CPF by shape or by arithmetic.

This is worse than an ordinary false negative, because it is **silent and systematic**. It does not depend on
awkward text, on OCR or on abbreviation. It depends on the window size, which is a decision made by whoever
built the indexing pipeline, and which is almost never revisited with personal data in mind.

## So where does redaction belong?

Before the split, on the whole document.

```{code-cell}
redacted = tarja.mask(document)
print(redacted)
print()
for c in [redacted[i:i + size] for i in range(0, len(redacted), size)]:
    print(f"  {c!r}")
```

No chunk carries personal data, and at ingestion time that is what matters.

Notice the side effect, visible in the output: the cut now splits the **token** rather than the number. One chunk
keeps `<BR_` and the other `CPF>`. Nothing leaked, because the token is not the value, but a split token cannot be
reversed by the `reveal` of chapter 5, which is one more reason `reveal` does not belong in this design.

The rule worth remembering: **redact before you split**, not the other way round.

If the pipeline you inherited already splits before any treatment, the stopgap is an overlapping window, so
that every identifier appears whole in at least one chunk:

```{code-cell}
overlap = 20
for c in [document[i:i + size + overlap] for i in range(0, len(document), size)]:
    print(f"{c!r:66} -> {[m.entity for m in tarja.find(c)]}")
```

It works, and it is a patch. The overlap has to exceed the longest identifier you expect, nobody checks that,
and storage cost goes up. Prefer redacting first.

## The question carries personal data too

Someone who asks "what does the case for CPF 529.982.247-25 say" has just sent a CPF to the same model, and
probably to the same log. There are **two places to apply redaction**, not one: ingestion and query.

```{code-cell}
question = "what does the case for CPF 529.982.247-25 say?"
print(tarja.mask(question))
```

The query side is the more forgotten of the two, because the team that handled ingestion considered the problem
solved. It is also the more frequent, because ingestion happens once and questions happen every day.

## Why does the pseudonym have to be stable here?

Because in an index, retrieval usefulness depends on the same identifier becoming the same token across
documents. If every document gets a different token for the same CPF, you can no longer retrieve everything
about one person, which is often the point of the query. This is `pseudonym_stable` from chapter 4.

```{code-cell}
import secrets

key = secrets.token_hex(32)
for d in ["Penalty issued against the holder of CPF 529.982.247-25.",
          "Appeal filed by the holder of CPF 529.982.247-25."]:
    print(tarja.mask(d, strategy="pseudonym_stable", salt=key))
```

The same token in both. That is what preserves the ability to join, without keeping the value.

## What this tool does not solve in that design

**The vault does not survive the index.** The `reveal()` of chapter 4 relies on a vault that is local to the
process, single use, and valid for one hour. That was chosen deliberately, so the map from token to value would
not become a database of personal data. A vector index lives for months. In practice: `mask` is right for
ingestion, and `reveal` will not give you the original value back on a query made weeks later. If your case
needs that, you need a key custodian of your own, with the lifecycle and the audit trail it implies, and that
is an architecture decision rather than a function call.

**The order is irreversible.** Redact and then embed, never the reverse, because there is no unredacting a
vector. If the index was built from raw text, the fix is to reindex.

**Deletion is an open problem.** An erasure request against a vector index is not a detection question, it is a
storage engineering one. Finding every chunk belonging to one person requires that you kept that link, which is
precisely what pseudonymisation avoids keeping. There is no clean answer here, and a book that pretended
otherwise would be worse than this one.

## Takeaway

RAG is not a special case of "sending text to a model". It is the case where the text stays, where the split
destroys the arithmetic that would have recognised the data, and where the question is a second leak that
almost nobody guards.

If after these eight chapters you have a clear sense of what you can and cannot claim, the book did what it set
out to do. The rest is measurement.

---

The instrument's code and documentation: [tarja](https://macmaia.github.io/tarja/).
Portuguese companion, framed around the LGPD:
[Python para Anonimizar Dados Pessoais](https://macmaia.github.io/Python-para-Anonimizar-Dados-Pessoais/).
