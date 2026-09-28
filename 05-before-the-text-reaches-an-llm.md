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
    description: "How to strip personal data from text before sending it to a language model, and how to put the original values back into the answer without storing them."
---

# 5. Before the text reaches a language model

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

This is the fastest growing case and the least handled. Somebody connects an assistant to the ticketing
system, or builds retrieval over the document store, and from that day on your customers' text leaves your
perimeter on every request, with whatever is inside it.

The previous chapter ended on three strategies. None of them fits here, because all three are one-way and this
case needs the value back: the model answers talking about the account holder, and the answer has to make sense
to whoever reads it.

## How do you redact before an LLM and restore after?

```{code-cell}
import tarja

text = "Patient CPF 529.982.247-25, health card 898 0000 0004 3208."

vault = tarja.Vault()
safe = vault.protect(text)
print(safe)
```

Each identifier became a token. The result is an ordinary string: it goes to the model's API, to a file or into
a retrieval index with no special handling.

The model works on that text and answers using the same tokens:

```{code-cell}
import re

cpf_token = re.search(r"<BR_CPF:[^>]+>", safe).group()
model_answer = f"The holder {cpf_token} should return in 30 days."
print(model_answer)
```

And you rehydrate:

```{code-cell}
print(vault.reveal(model_answer, issued_by=safe))
```

Four lines. The personal data did not leave, and the answer reached the user complete.

## Check, do not trust

Masking is easy to believe and hard to guarantee. An identifier the detector does not know stays in the text,
and the text has already been sent.

```{code-cell}
print("anything left in 'safe'?", tarja.residual(safe))
print("and here?", [m.entity for m in tarja.residual("a stray CPF 529.982.247-25 here")])
```

`residual()` is a second pass over the already-treated text. It ignores tokens and looks for real identifiers.
The working rule: run it before sending, and if it returns anything, do not send.

This does not prove the text is clean. It proves the detector finds nothing more. Those are different claims,
and chapter 7 insists on the difference.

## Why the token is bound to the call

```{code-cell}
other_vault = tarja.Vault()
try:
    other_vault.reveal(model_answer, issued_by=safe)
except Exception as e:
    print(type(e).__name__)
```

A vault does not resolve tokens it did not issue. That looks obvious, and the case that matters is subtler: in
a multi-user service one vault can serve many people. If `reveal()` resolved any token that vault had ever
issued, user A could paste into a text field a token that appeared in user B's document and get B's data back.

Hence the scope is bound to the `protect()` call that issued the tokens, and hence it is single use:

```{code-cell}
try:
    vault.reveal(model_answer, issued_by=safe)
except Exception as e:
    print(type(e).__name__)
```

The second attempt on the same scope is refused. Pass `reuse=True` when repetition is legitimate, knowing you
are giving up a control.

## The model mangles the token

This is the failure that shows up in production and not in testing.

A language model does not treat the token as an untouchable symbol. It wraps it across a line, changes the case
of the hex, sometimes paraphrases around it.

```{code-cell}
wrapped = cpf_token[:20] + "\n" + cpf_token[20:]
print(repr(wrapped))
print("resolves?", "529.982.247-25" in vault.reveal(f"Holder {wrapped}.", issued_by=safe, reuse=True))
```

It resolves. Whitespace and letter case carry no information, so accepting them costs **zero** in security: the
hex characters still have to match exactly, and a forger is no closer than before.

And that is where the tolerance stops. A token with a digit missing does not resolve:

```{code-cell}
truncated = cpf_token[:-3] + ">"
print("resolves?", "529.982.247-25" in vault.reveal(f"Holder {truncated}.", issued_by=safe, reuse=True))
```

Every extra character of tolerance is one fewer character of secret, and that is how an oracle gets built by
accident: somebody who can try often enough eventually hits a token that resolves.

Two things to do anyway. Instruct the model, in the prompt, to repeat tokens exactly as received, which reduces
the problem at source. And treat an answer that still contains a token after rehydration as an error on your
side, without showing it to the user.

## What the library does not do

The token-to-value map lives in the memory of one process. That has three consequences you need to know before
production, not after.

It dies with the process, so an ingestion job cannot hand tokens to a separately running service. It does not
cross replicas: with two instances behind a load balancer, `protect()` and `reveal()` for one document have to
land on the same one. And there is no record of who revealed what.

None of that is an oversight. Storing that map durably drags in key custody, access control, retention,
deletion on request, backups and audit, which are decisions about the operator's risk rather than a sensible
library default. Whoever holds the `Vault` object reverses everything, just like whoever holds the key.

## Takeaway

Tokenise, send, rehydrate, check the residue. Four lines for the most common route personal data takes out of a
company today. The scope bound to the call is what stops one user reversing another's data. And the model will
mangle a token now and then, which is a visible defect rather than a leak.

Next: measuring how much of this is actually working.
