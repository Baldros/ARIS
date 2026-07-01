# Legal Sources Architecture for ARIS

This document describes the recommended architecture for discovering, resolving, retrieving, cleaning, and indexing Brazilian legal texts in ARIS.

The main idea is simple:

```text
LexML = discovery + URN + metadata
URN = routing by normative type
Planalto = official federal legal text
DuckDuckGo = final fallback only
```

This should be treated as the baseline strategy for the legal-source layer of the agent.

---

## 1. Problem statement

ARIS needs to answer legal questions using reliable legal sources. For this, the system must not treat every result found on the web as equivalent.

A legal agent must distinguish between:

- current consolidated legal text;
- original publication text;
- bills and legislative proposals;
- case law;
- doctrine;
- books, articles, and secondary commentary;
- unofficial mirrors or legal blogs.

For Brazilian federal legislation, the safest practical strategy is to combine multiple sources, each with a specific responsibility.

---

## 2. Source roles

### 2.1 LexML: discovery, URN, and metadata

LexML should be used as the discovery layer.

Its role is to answer questions such as:

- Which legal documents are related to this topic?
- Which documents are actual laws, and which are bills, doctrine, or case law?
- What is the persistent URN for the document?
- What metadata is available for the document?

LexML is useful because it exposes structured identifiers such as:

```text
urn:lex:br:federal:lei:1990-09-11;8078
urn:lex:br:federal:decreto.lei:1940-12-07;2848
urn:lex:br:federal:lei.complementar:2026-01-08;225
urn:lex:br:senado.federal:projeto.lei;pls:1989;97
```

The URN is the main bridge between discovery and retrieval.

However, LexML should not be treated as the final source of the legal text. It is better used as a catalog, identifier source, and metadata layer.

### 2.2 URN: routing by normative type

The URN should be parsed and used to route the document to the correct resolver.

For example:

```text
urn:lex:br:federal:decreto.lei:1940-12-07;2848
```

Can be decomposed as:

```text
country: br
level: federal
type: decreto.lei
date: 1940-12-07
number: 2848
```

This tells ARIS that the document is a federal decree-law and should first be resolved against the decree-law paths in Planalto.

Another example:

```text
urn:lex:br:federal:lei:1990-09-11;8078
```

Can be decomposed as:

```text
country: br
level: federal
type: lei
date: 1990-09-11
number: 8078
```

This tells ARIS that the document is a federal ordinary law and should first be resolved against the ordinary-law paths in Planalto.

### 2.3 Planalto: official federal legal text

For federal legislation, Planalto should be the main source of the full legal text.

The Planalto URLs are not perfectly uniform, but they follow useful patterns under:

```text
https://www.planalto.gov.br/ccivil_03/
```

The `ccivil_03` directory is the practical root for many official federal legal texts maintained by the Presidency / Casa Civil / Subchefia para Assuntos Juridicos.

The more human-friendly portal is:

```text
https://www4.planalto.gov.br/legislacao
```

But for automation, the `ccivil_03` paths are more useful.

### 2.4 DuckDuckGo: final fallback

DuckDuckGo or any open web search should be used only as a final fallback.

The preferred order is:

```text
1. Resolve by known Planalto URL patterns
2. Resolve the LexML URN page and extract official links
3. Search within Planalto only
4. Use open web search as the final fallback
```

Open web search can return unofficial mirrors, legal blogs, PDFs, outdated pages, and commentary. It is useful as a recovery mechanism, not as the primary source of truth.

---

## 3. Recommended pipeline

The source-resolution pipeline should be:

```text
User query
  -> LexML search
  -> Extract candidate URNs
  -> Classify candidate URNs
  -> Prioritize actual legal norms
  -> Parse selected URN
  -> Route by normative type
  -> Generate Planalto candidate URLs
  -> Fetch first valid official URL
  -> Clean and normalize text
  -> Segment by article / paragraph / item
  -> Store metadata + raw HTML + clean text
  -> Index chunks for retrieval
```

For legal reliability, ARIS should keep raw source material and cleaned text separately.

---

## 4. Classification rules

LexML search results can include many document categories. ARIS should classify results before using them.

Recommended high-level categories:

```text
legal_norm
bill_or_proposal
case_law
doctrine
book_or_article
unknown
```

Examples:

```text
urn:lex:br:federal:lei:1990-09-11;8078
```

Should be classified as:

```text
legal_norm / federal_law
```

While:

```text
urn:lex:br:senado.federal:projeto.lei;pls:1989;97
```

Should be classified as:

```text
bill_or_proposal
```

The bill or proposal may be useful for legislative history, but it should not be treated as the current law.

Important rule:

```text
bill != law
doctrine != law
case law != statute
original publication != consolidated current text
```

---

## 5. Planalto base paths

The resolver should maintain a map of Planalto base paths.

```python
BASES_PLANALTO = {
    "constituicao": "https://www.planalto.gov.br/ccivil_03/constituicao/",
    "lei": "https://www.planalto.gov.br/ccivil_03/leis/",
    "lei_complementar": "https://www.planalto.gov.br/ccivil_03/leis/lcp/",
    "decreto": "https://www.planalto.gov.br/ccivil_03/decreto/",
    "decreto_lei": "https://www.planalto.gov.br/ccivil_03/decreto-lei/",
    "ato": "https://www.planalto.gov.br/ccivil_03/_ato{inicio}-{fim}/{ano}/",
}
```

This map is not enough by itself, but it is the right starting point.

---

## 6. URL candidate generation

There is no single universal Planalto URL pattern.

For example, this works for the Brazilian Consumer Defense Code:

```text
https://www.planalto.gov.br/ccivil_03/leis/l8078compilado.htm
```

But other laws may use:

```text
l10406compilada.htm
/leis/2002/l10406compilada.htm
/_ato2019-2022/2021/lei/L14133.htm
```

So ARIS should generate multiple candidates and test them in order.

### 6.1 Ordinary federal law

For:

```text
urn:lex:br:federal:lei:1990-09-11;8078
```

Generate candidates such as:

```text
https://www.planalto.gov.br/ccivil_03/leis/l8078compilado.htm
https://www.planalto.gov.br/ccivil_03/leis/l8078compilada.htm
https://www.planalto.gov.br/ccivil_03/leis/l8078.htm
https://www.planalto.gov.br/ccivil_03/leis/1990/l8078compilado.htm
https://www.planalto.gov.br/ccivil_03/leis/1990/l8078compilada.htm
https://www.planalto.gov.br/ccivil_03/leis/1990/l8078.htm
```

For newer laws, also try the presidential-period directory:

```text
https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2026/lei/l15388.htm
```

### 6.2 Complementary law

For:

```text
urn:lex:br:federal:lei.complementar:2026-01-08;225
```

Generate candidates such as:

```text
https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp225.htm
https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp225compilado.htm
https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp225compilada.htm
https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2026/lei/lcp225.htm
https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2026/lei/LCP225.htm
```

### 6.3 Decree-law

For the Penal Code:

```text
urn:lex:br:federal:decreto.lei:1940-12-07;2848
```

Generate candidates such as:

```text
https://www.planalto.gov.br/ccivil_03/decreto-lei/del2848compilado.htm
https://www.planalto.gov.br/ccivil_03/decreto-lei/del2848compilada.htm
https://www.planalto.gov.br/ccivil_03/decreto-lei/del2848.htm
```

For the Criminal Procedure Code:

```text
urn:lex:br:federal:decreto.lei:1941-10-03;3689
```

Generate candidates such as:

```text
https://www.planalto.gov.br/ccivil_03/decreto-lei/del3689compilado.htm
https://www.planalto.gov.br/ccivil_03/decreto-lei/del3689compilada.htm
https://www.planalto.gov.br/ccivil_03/decreto-lei/del3689.htm
```

### 6.4 Decree

For federal decrees, generate candidates such as:

```text
https://www.planalto.gov.br/ccivil_03/decreto/d{number}.htm
https://www.planalto.gov.br/ccivil_03/decreto/d{number}compilado.htm
https://www.planalto.gov.br/ccivil_03/decreto/d{number}compilada.htm
https://www.planalto.gov.br/ccivil_03/_ato{period}/{year}/decreto/d{number}.htm
```

---

## 7. Presidential-period resolver

Modern acts are often organized under presidential-period folders:

```text
/_ato1995-1998/
/_ato1999-2002/
/_ato2003-2006/
/_ato2007-2010/
/_ato2011-2014/
/_ato2015-2018/
/_ato2019-2022/
/_ato2023-2026/
```

The resolver should infer the correct period from the year contained in the URN.

Example:

```text
year = 2021
period = 2019-2022
```

Candidate path:

```text
https://www.planalto.gov.br/ccivil_03/_ato2019-2022/2021/lei/L14133.htm
```

---

## 8. Validation rules

Do not accept a URL only because it returned HTTP 200.

The resolver should validate at least:

- the domain is `planalto.gov.br`;
- the response is HTML or readable text;
- the response body is not too small;
- the text contains the expected number, title, or normative type;
- the document is not merely a generic error page;
- the fetched page is consistent with the URN metadata.

Recommended metadata checks:

```text
expected number
expected year
expected normative type
expected title when available
expected source domain
```

When validation is uncertain, store the document but mark it as low confidence.

---

## 9. Fallback strategy

Recommended fallback order:

```text
1. Planalto candidate URL resolver
2. LexML URN resolver page
3. Extract official links from LexML page
4. Search restricted to Planalto
5. Search restricted to official government domains
6. DuckDuckGo / open web search
```

Open web search should never silently override an official source.

When the final source is not official, ARIS should mark the result as:

```text
source_confidence = low
source_type = unofficial_fallback
```

---

## 10. Text treatment and normalization

Once the official text is fetched, ARIS needs a text-treatment pipeline.

The pipeline should keep both:

```text
raw_html
clean_text
```

Recommended stages:

```text
1. Fetch HTML
2. Store raw HTML
3. Detect encoding
4. Remove navigation, headers, footers, scripts, and styles
5. Preserve legal structure
6. Normalize whitespace
7. Normalize article markers
8. Segment by article, paragraph, item, subitem, and heading
9. Store segment metadata
10. Generate embeddings only after segmentation
```

Legal text should not be treated as ordinary prose. The structure matters.

Important markers to preserve:

```text
Art.
Paragrafo unico
§
I, II, III, IV...
a), b), c)...
CAPITULO
SECAO
TITULO
LIVRO
```

The chunking strategy should avoid splitting an article from its paragraphs unless necessary.

---

## 11. Suggested storage model

ARIS should store document metadata separately from text chunks.

### 11.1 `legal_documents`

Suggested fields:

```text
id
urn
type
level
number
year
date
title
nickname
summary
source_url
source_domain
source_kind
is_consolidated
retrieved_at
source_confidence
raw_metadata
```

### 11.2 `legal_document_versions`

Suggested fields:

```text
id
document_id
version_date
source_url
raw_html_path
clean_text_path
is_current
retrieved_at
```

### 11.3 `legal_text_chunks`

Suggested fields:

```text
id
document_id
version_id
chunk_index
heading_path
article
paragraph
item
subitem
text
source_url
embedding
```

This allows ARIS to answer questions while citing the exact legal text segment.

---

## 12. Current text vs historical text

A legal agent must distinguish between:

```text
current consolidated text
original publication text
historical version at a specific date
```

For most first-pass user queries, ARIS should prefer the current consolidated text.

However, if the user asks something like:

```text
What did the law say in 2018?
```

Then ARIS must retrieve or reconstruct the historical version. If this is not available, the answer should explicitly say that only the current or original text was found.

---

## 13. Example: Consumer Defense Code

LexML search for `Codigo do Consumidor` may return:

```text
urn:lex:br:federal:lei:1990-09-11;8078
urn:lex:br:senado.federal:projeto.lei;pls:1989;97
```

The first one is the actual law:

```text
urn:lex:br:federal:lei:1990-09-11;8078
```

It should be routed as:

```text
type = lei
number = 8078
year = 1990
resolver = ordinary federal law resolver
```

The second one is a legislative proposal:

```text
urn:lex:br:senado.federal:projeto.lei;pls:1989;97
```

It may be useful for legislative history, but it should not be treated as the law itself.

---

## 14. Example: Penal Code

LexML search for `Codigo Penal` may return:

```text
urn:lex:br:federal:decreto.lei:1940-12-07;2848
urn:lex:br:federal:decreto.lei:1969-10-21;1001
urn:lex:br:federal:decreto.lei:1941-10-03;3689
```

The Penal Code is:

```text
urn:lex:br:federal:decreto.lei:1940-12-07;2848
```

It should be routed as:

```text
type = decreto.lei
number = 2848
year = 1940
resolver = federal decree-law resolver
```

Candidate Planalto URL:

```text
https://www.planalto.gov.br/ccivil_03/decreto-lei/del2848compilado.htm
```

---

## 15. Implementation sketch

```python
import re
from dataclasses import dataclass


@dataclass
class NormaUrn:
    level: str
    normative_type: str
    date: str
    year: int
    number: str


def parse_lexml_urn(urn: str) -> NormaUrn:
    pattern = (
        r"^urn:lex:br:"
        r"(?P<level>federal|estadual|municipal|distrital):"
        r"(?P<type>[^:]+):"
        r"(?P<date>\d{4}-\d{2}-\d{2});"
        r"(?P<number>[\w\.-]+)$"
    )

    match = re.match(pattern, urn)

    if not match:
        raise ValueError(f"Unsupported LexML URN: {urn}")

    date = match.group("date")

    return NormaUrn(
        level=match.group("level"),
        normative_type=match.group("type"),
        date=date,
        year=int(date[:4]),
        number=match.group("number").replace(".", ""),
    )
```

```python
def presidential_period(year: int) -> tuple[int, int] | None:
    periods = [
        (1995, 1998),
        (1999, 2002),
        (2003, 2006),
        (2007, 2010),
        (2011, 2014),
        (2015, 2018),
        (2019, 2022),
        (2023, 2026),
    ]

    for start, end in periods:
        if start <= year <= end:
            return start, end

    return None
```

```python
def generate_planalto_candidates(norma: NormaUrn) -> list[str]:
    n = norma.number
    year = norma.year
    t = norma.normative_type

    urls = []

    if t == "lei":
        urls.extend([
            f"https://www.planalto.gov.br/ccivil_03/leis/l{n}compilado.htm",
            f"https://www.planalto.gov.br/ccivil_03/leis/l{n}compilada.htm",
            f"https://www.planalto.gov.br/ccivil_03/leis/l{n}.htm",
            f"https://www.planalto.gov.br/ccivil_03/leis/{year}/l{n}compilado.htm",
            f"https://www.planalto.gov.br/ccivil_03/leis/{year}/l{n}compilada.htm",
            f"https://www.planalto.gov.br/ccivil_03/leis/{year}/l{n}.htm",
        ])

        period = presidential_period(year)
        if period:
            start, end = period
            urls.extend([
                f"https://www.planalto.gov.br/ccivil_03/_ato{start}-{end}/{year}/lei/l{n}.htm",
                f"https://www.planalto.gov.br/ccivil_03/_ato{start}-{end}/{year}/lei/L{n}.htm",
                f"https://www.planalto.gov.br/ccivil_03/_ato{start}-{end}/{year}/lei/l{n}compilado.htm",
                f"https://www.planalto.gov.br/ccivil_03/_ato{start}-{end}/{year}/lei/l{n}compilada.htm",
            ])

    elif t == "lei.complementar":
        urls.extend([
            f"https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp{n}.htm",
            f"https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp{n}compilado.htm",
            f"https://www.planalto.gov.br/ccivil_03/leis/lcp/lcp{n}compilada.htm",
        ])

        period = presidential_period(year)
        if period:
            start, end = period
            urls.extend([
                f"https://www.planalto.gov.br/ccivil_03/_ato{start}-{end}/{year}/lei/lcp{n}.htm",
                f"https://www.planalto.gov.br/ccivil_03/_ato{start}-{end}/{year}/lei/LCP{n}.htm",
            ])

    elif t == "decreto.lei":
        urls.extend([
            f"https://www.planalto.gov.br/ccivil_03/decreto-lei/del{n}compilado.htm",
            f"https://www.planalto.gov.br/ccivil_03/decreto-lei/del{n}compilada.htm",
            f"https://www.planalto.gov.br/ccivil_03/decreto-lei/del{n}.htm",
        ])

    elif t == "decreto":
        urls.extend([
            f"https://www.planalto.gov.br/ccivil_03/decreto/d{n}.htm",
            f"https://www.planalto.gov.br/ccivil_03/decreto/d{n}compilado.htm",
            f"https://www.planalto.gov.br/ccivil_03/decreto/d{n}compilada.htm",
        ])

        period = presidential_period(year)
        if period:
            start, end = period
            urls.append(
                f"https://www.planalto.gov.br/ccivil_03/_ato{start}-{end}/{year}/decreto/d{n}.htm"
            )

    return urls
```

---

## 16. Operational principles

ARIS should follow these principles:

1. Prefer official sources.
2. Never treat all web results as equivalent.
3. Separate legal norms from bills, case law, and doctrine.
4. Store raw source material before cleaning.
5. Keep source URL and retrieval timestamp for every document.
6. Prefer consolidated text for general legal questions.
7. Use original publication for historical verification.
8. Use open web search only as a final fallback.
9. Mark fallback confidence explicitly.
10. Do not answer as if a source is official when it is not.

---

## 17. Summary

The recommended source architecture is:

```text
LexML search
  -> URN extraction
  -> URN classification
  -> Planalto resolver
  -> official text retrieval
  -> text treatment
  -> legal-structure-aware chunking
  -> retrieval index
  -> answer with citation
```

The short version is:

```text
LexML = discovery + URN + metadata
URN = routing by normative type
Planalto = official federal text
DuckDuckGo = final fallback
```

This architecture is robust enough for the first version of ARIS, while leaving room for future improvements such as historical-version retrieval, state and municipal legislation, DOU validation, and richer legislative-history analysis.
