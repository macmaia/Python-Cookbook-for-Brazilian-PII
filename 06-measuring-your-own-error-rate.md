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
    description: "How to measure your own error rate in personal data detection, with sampling, confidence intervals and precision and recall computed on your own corpus."
---

# 6. Measuring your own error rate

```{code-cell}
:tags: [skip-execution]
%pip install -q tarja
```

So far the book has shown what to do. This chapter is about knowing whether it worked on **your** text, which
is a different question from whether it works on the text of whoever wrote the tool.

## The two errors do not cost the same

A detector is wrong in two ways. It flags what is not, and it misses what is.

**A false positive** is the ticket number marked as a taxpayer number. The immediate cost is destroying
operational data, if you mask it. The larger cost is chapter 1's: once the list is noisy enough, nobody trusts
it, including the part that was right.

**A false negative** is the identifier still sitting in the text after you said you had treated it. The cost is
exposed personal data, and the aggravating factor is that you do not know it is there. A false positive report
is annoying. A false negative is what turns up in the incident.

Which one to tighten depends on what you are doing. Masking before sending text outside, a false negative is
worse and noise is worth accepting. Producing an inventory for a data protection officer to prioritise from, a
false positive is worse, because a wrong map leads to a wrong decision.

No configuration minimises both. You choose which error you prefer to make.

## What do precision and recall mean here?

**Precision** answers: of what I flagged, how much was right. **Recall** answers: of what was there, how much
did I find.

```{code-cell}
def evaluate(gold: set, pred: set) -> dict:
    tp = len(gold & pred)
    fp = len(pred - gold)
    fn = len(gold - pred)
    precision = tp / (tp + fp) if tp + fp else 0.0
    recall = tp / (tp + fn) if tp + fn else 0.0
    f1 = 2 * precision * recall / (precision + recall) if precision + recall else 0.0
    return {"TP": tp, "FP": fp, "FN": fn, "precision": round(precision, 3),
            "recall": round(recall, 3), "F1": round(f1, 3)}

# gold: what you annotated by hand. pred: what the tool found.
gold = {(13, 27, "BR_CPF"), (40, 58, "BR_CNS")}
pred = {(13, 27, "BR_CPF"), (70, 81, "BR_CPF")}
print(evaluate(gold, pred))
```

A hit counts as correct only if **both the type and the span** match. Getting the position right and the type
wrong is an error, because the treatment depends on the type.

## How do you sample and annotate your own corpus?

Precision requires knowing the right answer, and the right answer is not free. Somebody has to look and judge.

The shortcut most people take is to run the tool over their corpus, count the hits and present that as a
result. That measures nothing: it is the tool grading itself.

The honest procedure is short and tedious:

1. Run detection over your corpus and keep **every** hit.
2. Draw a random sample, with a fixed seed so you can repeat it.
3. Look at them one at a time, with surrounding text, and answer: is this really an identifier of that type?
4. Precision is the proportion of yes, with a confidence interval.

A sample of a hundred takes one to two hours and is the only number you can defend. Two hundred verified hits
are worth more than forty thousand unverified ones.

Note that this measures **precision** and not recall. Recall needs the opposite: take documents and annotate
them completely, from scratch, without looking at the tool's output. That is considerably more expensive, which
is why most vendor reports quote precision only, without mentioning they are quoting half.

## Why report a confidence interval?

```{code-cell}
def wilson(hits: int, total: int, z: float = 1.96) -> tuple[float, float]:
    """Wilson interval: honest at small n and near 0 or 1, unlike the textbook one."""
    if not total:
        return (0.0, 0.0)
    p = hits / total
    d = 1 + z * z / total
    centre = (p + z * z / (2 * total)) / d
    half = z * ((p * (1 - p) / total + z * z / (4 * total * total)) ** 0.5) / d
    return (max(0.0, centre - half), min(1.0, centre + half))

for hits, total in [(39, 39), (36, 39), (95, 100)]:
    lo, hi = wilson(hits, total)
    print(f"{hits}/{total}: precision {hits/total:.3f}  95% CI [{lo:.3f}, {hi:.3f}]")
```

Look at the first line. Thirty-nine correct out of thirty-nine gives a precision of 1.000, and the classic
normal approximation would give an interval of 1.000 to 1.000: absolute certainty from thirty-nine
observations, which is plainly false. Wilson says 0.910 to 1.000, which is what the data support.

That is why it is used here. Near zero or one, with a small sample, the normal approximation errs in the most
convenient direction.

When somebody presents a precision figure without an interval, the question is: over how many judgements.

## Suspects: right shape, wrong digit

There is a third category, and it is neither a false positive nor a false negative.

```{code-cell}
import tarja

for m in tarja.find("cpf 529.982.247-24", report_invalid=True):
    print(m.entity, "check digit valid:", m.valid_dv, "score:", m.score)
```

A number shaped like a taxpayer number whose digit does not close. It is not reported as a hit, so it never
enters prevalence. But it is rarely a coincidence: it is a real identifier with a digit typed wrong, or read
wrong by optical character recognition.

Two consequences. A high volume of suspects points at a problem upstream, which is useful diagnosis. And **a
suspect is near-personal-data**: it re-identifies the same person most of the time, because one wrong digit
leaves ten candidates and the surrounding text settles it.

So a suspect does not go into a log, an error message or a monitoring event. Count it, or carry it without the
value:

```{code-cell}
hit = tarja.find("Patient CPF 529.982.247-25")[0]
print(hit.to_dict(include_value=False))
```

And printing a hit is safe, because the value does not appear:

```{code-cell}
print(tarja.find("Patient CPF 529.982.247-25"))
```

That behaviour exists precisely because "do not log it" is easy to write in documentation and easy to forget in
code.

## Measure again, periodically

A measurement is a snapshot. Your text changes: a new system, a new format, a new supplier, a field that
started being filled in differently. Precision measured in March says nothing about October.

Repeat the sampling when the data source changes, when you upgrade the tool, and otherwise once every six
months. Record the seed and the sample size next to the number, or you cannot compare with the previous one.

## Takeaway

False positives and false negatives cost different things, and you choose which to make. Precision only exists
if somebody looked at and judged a sample. A confidence interval is not statistical decoration, it is what
stops you claiming certainty from thirty-nine observations. And a suspect is near-personal-data, not debug
output.

The last chapter is about what none of this does.
