---
layout: post
title: "When the BOJA won't parse"
date: 2026-04-30
lang: en
permalink: /boja-parser/
---

**TL;DR:** The BOJA published financial declarations for 1,129 candidates in a hard-to-parse PDF. I wrote a Python script to extract and structure the data, and a web app to browse it. [Explore →](https://tomaspica.com/declaraciones-andalucia-2026/) · [Code →](https://github.com/t00mas/declaraciones-andalucia-2026)

---

On April 27, 2026, the BOJA published issue 79 C1: 402 pages of financial declarations from the 1,129 candidates running in the May 17 regional elections. Public data, theoretically accessible to any citizen. In practice, a PDF.

Not just any PDF. One assembled from scanned forms and reformatted, with columns that `pdftotext` extracts in the wrong order, BOJA headers interleaved with candidate data, and entire sections displaced several pages from where they should be.

## The `pdftotext` problem

`pdftotext` converts PDFs to plain text following the visual column order of the page. For BOJA's tables, that's a disaster: some sections come out row-major (all fields for row 1, then row 2...), but others come out column-major (all values for column 1 across all rows, then all values for column 2). There's no way to know which is which without inspecting the original PDF layout.

The fix was to detect the pattern: if a section's text starts with a column header followed by all its values before the next header appears, it's column-major. If values from different columns interleave, it's row-major. The parser applies a different strategy for each case.

## BOJA noise

Between every page, the PDF injects BOJA footer fragments: legal deposit number, regional government URL, page number, the word "BOJA" alone on a line. All of it interleaved with candidate data. These need to be stripped without deleting real data that might look similar.

```python
def clean_headers(text):
    text = re.sub(r"Depósito Legal:[^\n]+\n\s*\n?\s*https?://[^\n]+\n?", "\n", text)
    text = re.sub(r"(?:BOJA\s*\n\s*)?Boletín Oficial de la Junta de Andalucía\s*\n[^\n]+\n\s*página \d+/\d+", "", text)
    text = re.sub(r"^\s*BOJA\s*$", "", text, flags=re.MULTILINE)
    # ...
    return text
```

## Displaced rows and out-of-place sections

The most common issue: `pdftotext` drops content from one section into another section's text. Section 2.3 (stocks and securities) can contain rows that actually belong in 2.4 (vehicles) or 2.6 (debts), because the original PDF had them close together and the extraction merged them.

Detection heuristic: if the text of section 2.3 contains a "Descripción" header (which belongs to section 2.4), there's displaced content. Re-parse it with the correct column format and reclassify each row by keyword: "hipoteca" → debt, "seguro de vida" → insurance, rest → vehicles.

```python
_seg_kw = re.compile(r"(?i)(segur|póliza)")
_deu_kw = re.compile(r"(?i)(hipoteca|préstamo|crédito|financiaci)")

if re.search(r"(?m)^Descripción\s*$", sec23_text):
    displaced_rows = parse_table("2.4", sec23_text)
    for row in displaced_rows:
        desc = str(row.get("Descripción") or "")
        if _seg_kw.search(desc):
            seguros.append(row)
        elif _deu_kw.search(desc):
            deudas.append(row)
        else:
            vehiculos.append(row)
```

## Manual overrides

For some candidates the layout is irregular enough that no heuristic gets it right. The acting president of the Junta, Juan Manuel Moreno Bonilla, has two rural plots in Cártama whose cadastral value is fused with the location description in a single text block. The parser detects it but can't separate it cleanly. Solution: manual override for that candidate.

## The result

A `candidates.json` file with 1,129 structured objects: calculated financials (net worth, total assets, total debt) plus complete detail tables.

On top of that JSON, a single-page web app with no server dependencies and no build step: a searchable, filterable table, a party-by-party wealth comparison, alert flags for notable financial patterns, and a scatter plot of all candidates colored by party.

**[→ Explore the declarations](https://tomaspica.com/declaraciones-andalucia-2026/)** · **[→ Repository](https://github.com/t00mas/declaraciones-andalucia-2026)**

Code is MIT. Data is public (BOJA).
