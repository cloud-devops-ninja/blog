---
layout: post
title:  "Parsing the CIS Benchmark PDF into Structured JSON: A Regex Adventure"
date:   2026-05-20 12:00:00 +0100
authors: [cdo-ninja]
image: assets/images/posts/PDF2JSONConverter.png
tags: [blog]
hidden: false
toc: false
---
## How I turned a 300-page security benchmark document into a machine-readable database the AI agent can use ... with a lot of help from Claude Code

---

## Why this problem exists

The CIS Windows 11 Benchmark v4.0 is a 300-page PDF. It is the canonical reference for Windows hardening — every major endpoint management platform, auditor, and security team uses it. But it exists as a document for humans, not a database for machines.

To build a [compliance agent](https://www.cloud-devops.ninja/posts/2026/05/19/IntuneCISCompliantAgent.html) that could compare Intune configuration profiles against CIS recommendations, I needed the benchmark data in a structured format: benchmark ID, setting name, recommended value, severity, description, remediation steps, and the exact Intune Settings Catalog path to configure it. None of that exists as a public API or pre-built dataset. The only authoritative source is the PDF.

So with a lot of help from Claude Code I created `extract_benchmarks.py` — a one-off script that reads `CIS_Microsoft_Intune_for_Windows_11_Benchmark_v4.0.0.pdf` and produces `benchmarks.json`. a JSON formatted list of all the CIS Benchmark recommendations from the PDF file. What looked like a simple, two-hour task took considerably longer. Here is what I ran into.

---

## The approach: extract, split, parse

The script follows a three-stage pipeline:

1. **Extract** — read all PDF pages into a single string using `pypdf`
2. **Split** — locate every benchmark entry in the text using an anchor regex and carve out per-benchmark blocks
3. **Parse** — pull structured fields out of each block and build the final JSON

```python
from pypdf import PdfReader

r = PdfReader("CIS_Microsoft_Intune_for_Windows_11_Benchmark_v4.0.0.pdf")
full_text = "\n".join(page.extract_text() or "" for page in r.pages)
```

Simple enough in concept. The complications were entirely in stages 2 and 3.

---

## Challenge 1: The table of contents echoes every benchmark title

The PDF has a table of contents that spans roughly twenty pages. It lists every benchmark by section number and name. When you extract the full text of the document, those titles appear twice — once in the ToC, once in the actual benchmark section.

This matters because I needed to split the document into per-benchmark blocks. Claude Code helped me use the following anchor pattern:

```python
ID_LEVEL_RE = re.compile(r"(\d+(?:\.\d+)+)\s+\((L1|L2|BL|NG)\)\s+Ensure\s+")
```

This matches lines like `1.1 (L1) Ensure 'Allow Cortana Above Lock Screen' is set to 'Disabled'`. The problem: the ToC contains exactly this pattern for every benchmark, so `finditer` returned double the expected matches — one from the ToC entry and one from the actual content.

The filter was simple but non-obvious: every real benchmark section contains the string `"Profile Applicability:"`, which marks the start of the structured body. ToC lines do not. So after splitting on the anchor, Claude Code suggested to discard any block that lacked this string:

```python
if "Profile Applicability:" in block:
    blocks.append((bench_id, level, block))
```

This cut the block count exactly in half and left only genuine benchmark entries.

---

## Challenge 2: Page headers and footers bleed into the text

PDF text extraction does not know about document structure — it knows about character positions on a page. Headers and footers land in the extracted text wherever they happen to fall in reading order, which is often mid-sentence or mid-field.

The CIS PDF has page numbers that appear in the extracted text as isolated lines like `\nPage 47\n` in the middle of a description paragraph. Once again Claude Code came to the rescue with a method to strip the page numbers before any further processing:

```python
def clean(text: str) -> str:
    text = re.sub(r"\nPage \d+\s*\n", " ", text)
    text = re.sub(r"\s+", " ", text)
    return text.strip()
```

The second substitution collapses all remaining whitespace — tabs, multiple spaces, newlines — into a single space. This is destructive (it loses formatting), but since I was extracting fields into flat strings anyway, it was the right trade-off for this use-case.

---

## Challenge 3: Extracting setting name and recommended value from the title

Each benchmark title follows a pattern like:

> Ensure 'Allow Cortana Above Lock Screen' is set to 'Disabled'

The setting name is in single quotes, and the recommended value is in the second quoted phrase. But not every title follows this pattern. Some read:

> Ensure 'Windows Firewall: Domain: Firewall state' is set to 'On (recommended)'

And others are more irregular:

> Ensure 'Minimum password age' is set to '1 or more day(s)'

Luckily Claude Code found an elegant way for the extractor to handles this in two passes — first try to match the quoted pattern, then fall back to the raw title:

```python
def extract_setting_and_recommendation(title_block: str):
    title = clean(title_block)
    title = re.sub(r"\s*\((Automated|Manual)\)\s*$", "", title)
    setting_m = re.search(r"'(.+?)'", title)
    setting = setting_m.group(1) if setting_m else clean(title)[:80]
    rec_m = re.search(r"is set to '(.+?)'", title)
    if not rec_m:
        rec_m = re.search(r"is set to (.+?)$", title)
    recommendation = rec_m.group(1).strip().rstrip(".") if rec_m else ""
    return setting, recommendation
```

The `(Automated)` / `(Manual)` suffix that CIS appends to some titles also needed stripping — it is metadata about the audit method, not part of the setting name.

---

## Challenge 4: Multi-line CSP paths broken by PDF layout

The remediation section of each benchmark contains the Intune Settings Catalog navigation path. In the source document this looks clean:

> **Settings Catalog path:** Above Lock > Allow Cortana Above Lock

But after PDF text extraction, long paths are broken across lines wherever the PDF renderer wrapped them, with no consistent delimiter. A path like:

```text
Administrative Templates\MS Security Guide\Enable Structured Exception
Handling Overwrite Protection (SEHOP)
```

comes out as two lines that need to be joined. Worse, some paths continue across a page boundary, which means a stray page number sits in the middle.

Claude Code suggestion was to have the CSP extractor use a regex that captures everything between the `"Settings Catalog path to"` marker and the next known field boundary, then have it join the lines:

```python
def extract_csp(remediation_text: str) -> str:
    m = re.search(
        r"Settings Catalog path to .+?\.\s*\n((?:(?!Default Value:|References:|Audit:|Impact:|Rationale:|Profile).+\n?)+)",
        remediation_text,
        re.DOTALL,
    )
    if m:
        lines = [l.strip() for l in m.group(1).splitlines() if l.strip()]
        path_lines = []
        for line in lines:
            if re.match(r"Note:", line, re.IGNORECASE):
                break
            path_lines.append(line)
        return " \\ ".join(path_lines).rstrip(".")
    return ""
```

The `" \\ ".join(path_lines)` reconstructs the backslash-separated hierarchy from whatever the PDF broke into separate lines. The `Note:` stop condition prevents footnotes that immediately follow the path from being absorbed into it.

---

## Challenge 5: Not all benchmarks can be configured through the UI

Some CIS controls apply to settings that Intune cannot configure through the Settings Catalog or Administrative Templates. They require either a direct registry OMA-URI or a PowerShell script. The PDF says so explicitly in the remediation text: *"This setting is not possible through Settings Catalog"*.

For these, Claude Code added the following code so the remediation steps builder detects the signal and changes the output entirely:

```python
is_powershell_only = "not possible through Settings Catalog" in text or (
    "PowerShell" in text and not is_settings_catalog
)

if is_powershell_only:
    steps.append(
        "Remediation not possible via Settings Catalog or OMA-URI; "
        "deploy via Intune Scripts or Remediations blade"
    )
```

Rather than generating a Settings Catalog navigation path that does not exist, the tool tells the agent (and the user) the honest truth: this one needs a script.

---

## Challenge 6: Generating useful remediation steps, not raw PDF text

The raw PDF remediation sections are verbose. They contain audit procedures, default value explanations, references to Group Policy paths, and sometimes two or three paragraphs of context. None of that is useful to someone who just wants to know where to click in Intune.

Here Claude Code suggested code so the `build_remediation_steps` function synthesizes a four-to-five step list tailored to the profile type:

```python
def build_remediation_steps(remediation_text, csp, setting, recommendation):
    steps = []
    is_settings_catalog = "Settings Catalog" in text
    is_admin_templates = "Administrative Templates" in (csp or "")
    is_endpoint_security = any(
        kw in (csp or "").lower()
        for kw in ["firewall", "defender", "antivirus", "exploit", "bitlocker"]
    )

    # Step 1 — where to navigate in Intune
    if is_admin_templates:
        steps.append("In Intune: Devices > Configuration profiles > Create > "
                     "Windows 10 and later > Administrative Templates")
    elif is_settings_catalog:
        steps.append("In Intune: Devices > Configuration profiles > Create > "
                     "Windows 10 and later > Settings Catalog")
    elif is_endpoint_security:
        steps.append("In Intune: Endpoint Security > select the relevant policy type")

    # Step 2 — what to configure
    if csp:
        steps.append(f"Navigate to and configure: {csp}")

    # Step 3 — value to set
    if recommendation and csp:
        steps.append(f"Set the value to: {recommendation}")

    # Step 4 — any PDF notes
    note_m = re.search(r"Note:\s*(.+?)(?:Default Value:|References:|$)",
                       remediation_text, re.DOTALL | re.IGNORECASE)
    if note_m:
        steps.append(f"Note: {clean(note_m.group(1))[:200]}")

    # Step 5 — assign
    if steps and not is_powershell_only:
        steps.append("Assign the policy to the target Windows 11 device groups")
```

The distinction between Settings Catalog, Administrative Templates, and Endpoint Security is important for the agent: these are three different places in the Intune portal, and a user following the wrong navigation path will not find the setting.

---

## Challenge 7: Mapping section numbers to category names

The JSON output is keyed by category name, not by section number. `"Above Lock"`, `"Administrative Templates"`, `"Credential Guard"`, and so on. The mapping from section number (`1`, `4`, `9`…) to human-readable name came from the ToC.

Parsing the ToC reliably was its own small problem. The table of contents lines look like:

```text
1   Above Lock .......... 42
4   Administrative Templates .......... 87
```

But the dots and page numbers vary in format. Fortunately for me Claude Code is also skilled in regex and produced the regex that captures them:

```python
toc_re = re.compile(
    r"^[ \t]*(\d{1,3})[ \t]+([A-Za-z][A-Za-z0-9 /\(\)\-]+?)[ \t]*(?:\.{3,}|[ \t]+\d+[ \t]*$)",
    re.MULTILINE,
)
```

This matches a leading number, then a name that starts with a letter and contains alphanumeric characters and common punctuation, followed by either a dotted leader (`......`) or a plain number at the end of the line. The `"ensure" not in name.lower()` guard drops individual benchmark titles that happen to match the pattern.

---

## The result

Running the script on the 300-page PDF takes about 30 seconds and produces a structured JSON file grouped by category:

```text
Reading PDF...
  323 pages, 1,247,831 chars
  Mapped 18 top-level sections

  Found 246 benchmark blocks

Parsed 246 benchmarks across 18 categories
  (14 PowerShell-only / no CSP path)

Category breakdown:
  Above Lock: 1
  Administrative Templates: 47
  Credential Guard: 4
  Firewall: 24
  ...

Wrote benchmarks.json (187 KB)
```

246 benchmarks, 18 categories, 187 KB. Every entry has a structured id, setting name, recommendation, severity, description, CSP path, and a list of concrete Intune navigation steps.

```json
{
  "Administrative Templates": [
    {
      "id": "4.1.3.1",
      "setting": "Prevent enabling lock screen camera",
      "recommendation": "Enabled",
      "severity": "High",
      "description": "Disables the lock screen camera toggle switch in PC Settings and prevents a camera from being invoked on the lock screen.",
      "csp": "Administrative Templates\\Control Panel\\Personalization\\Prevent enabling lock \\ screen camera",
      "remediation": [
        "In Intune: Devices > Configuration profiles > Create > Windows 10 and later > Administrative Templates",
        "Navigate to and configure: Administrative Templates\\Control Panel\\Personalization\\Prevent enabling lock \\ screen camera",
        "Set the value to: Enabled",
        "Assign the policy to the target Windows 11 device groups"
      ]
    }
  ]
}
```

---

## What the agent does with it

The benchmark database is loaded once at container startup and held in memory:

```python
class CISBenchmarkDatabase:
    BENCHMARKS = json.loads(_BENCHMARKS_FILE.read_text(encoding="utf-8"))
```

The agent has three tools for querying it:

- `get_cis_benchmarks(category)` — all controls in a category
- `search_cis_benchmarks(query)` — keyword search across all setting names
- `assess_compliance_status(benchmark_id)` — full detail and remediation for one control

No network call, no external API. The LLM gets structured, pre-processed data it can reason over directly — rather than being asked to interpret raw PDF prose in context.

---

## Lessons learned

**Claude Code's reasoning rocks, it sure helped this human-in-the-loop understand and validate the code.** I really enjoy my first Claude Code experience as it not just speed up my code production, but gave me a great learning experience by following its reasoning to not just understand the code but also validate the code and the JSON output. So I encourage you to not just copy and paste the code, but ask your prefered Model/Agent/AI to create your own version of the script (and have fun watching its reasoning and iterations as it tackles the challenges it faces)

**PDFs are not documents, they are drawing instructions.** Text extractors reconstruct reading order from character positions, which means anything the PDF author relied on visually (line breaks, columns, indentation) may not survive extraction intact. Budget time for this.

**Anchor-based splitting beats page-by-page processing.** Trying to process the PDF page by page would have been fragile — benchmark entries frequently span pages. Finding a reliable anchor that marks the start of each benchmark and slicing the full text string was simpler and more robust.

**Generate output for the consumer, not for the document.** The raw PDF remediation text is written for a human reading a PDF. The JSON output should be written for an LLM reasoning about Intune. Those are different audiences with different needs, and the gap between them is where most of the parsing logic lives.

**One-off scripts deserve real engineering.** This script only runs once per benchmark version. But because it was the foundation the entire agent was built on, errors in it would silently produce wrong compliance answers. Investing in the regex quality, the fallbacks, and the edge case handling was worth it.
