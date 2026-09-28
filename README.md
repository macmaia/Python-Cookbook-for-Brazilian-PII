# Python Cookbook for Brazilian PII

An open book about finding and handling Brazilian personal identifiers in text, for engineers whose PII
tooling does not know what a CPF is.

Read it at **https://macmaia.github.io/Python-Cookbook-for-Brazilian-PII/**

Companion book for a Brazilian audience, framed around the LGPD:
[Python para Anonimizar Dados Pessoais](https://github.com/macmaia/Python-para-Anonimizar-Dados-Pessoais).
It is not a translation of this one.

The instrument used throughout is [tarja](https://github.com/macmaia/tarja), Apache 2.0.

## build locally

```bash
pip install -r requirements.txt
jupyter-book build .
open _build/html/index.html
```

Code cells run at build time, so the output printed on each page is real. A chapter that breaks fails the
build, which is the behaviour you want in a book that teaches people to run code.

## licence

Text under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Code under Apache 2.0.

By [Maria Alice Maia](https://github.com/macmaia) · tarja@micah6ai.com
