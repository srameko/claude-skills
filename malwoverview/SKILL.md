---
name: malwoverview
description: >
  Static malware analysis workflow using malwoverview. Use this skill whenever
  the user wants to analyze a suspicious file, hash, or sample — including PE
  executables, DLLs, scripts (PowerShell, VBScript, JavaScript), Office documents,
  or PDFs. Also trigger when the user mentions malwoverview, IOC extraction,
  static analysis, strings analysis, entropy, YARA, or PE headers. Always apply
  OpSec skill PAP constraints before any action. Always use this skill when the
  user shares a file path, hash, or sample for analysis — even if they don't
  explicitly mention malwoverview.
---

# Malwoverview Skill

## Dependencies

- malwoverview installed at `~/venv/malwoverview`
- Activate venv: `source ~/venv/malwoverview/bin/activate`
- API: Ollama (primary, local, PAP-safe) → Anthropic (only if user explicitly requests + PAP permits)
- **Always apply OpSec skill PAP constraints before any action**

---

## Workflow

### Step 1: PAP Check
Before anything else, confirm PAP classification with the user or apply default
per OpSec skill. PAP level determines which modules are permitted.

### Step 2: Identify File Type
Determine the sample type to select appropriate modules:

```bash
file <sample>
```

| Type | Indicators |
|---|---|
| PE (EXE/DLL) | MZ header, `file` reports PE32/PE32+ |
| Script | .ps1, .vbs, .js, .bat, .hta extensions |
| Office document | .doc, .docx, .xls, .xlsm, .pptx, OLE/OOXML |
| PDF | %PDF header, .pdf extension |
| Archive/dropper | .zip, .rar, .7z, .iso, .img |

### Step 3: Select Modules

#### PE (EXE/DLL)
```bash
# Hash + VT lookup (only if PAP:GREEN or WHITE)
malwoverview.py -f <sample> -V 1

# Strings extraction
malwoverview.py -f <sample> -s 1

# PE headers + imports/exports
malwoverview.py -f <sample> -e 1

# Entropy analysis
malwoverview.py -f <sample> -E 1

# YARA rules
malwoverview.py -f <sample> -y <rules.yar>
```

#### Scripts (PS1, VBS, JS, BAT)
```bash
# Strings + deobfuscation hints
malwoverview.py -f <sample> -s 1

# Hash lookup (PAP permitting)
malwoverview.py -f <sample> -V 1
```

#### Office Documents
```bash
# OLE/macro extraction
malwoverview.py -f <sample> -f 1

# Strings
malwoverview.py -f <sample> -s 1
```

#### PDF
```bash
# Structure analysis + embedded objects
malwoverview.py -f <sample> -P 1

# Strings
malwoverview.py -f <sample> -s 1
```

### Step 4: API Enrichment
Use AI to interpret raw malwoverview output and enrich findings.

**Ollama (primary — local, PAP-safe at any level):**
```bash
malwoverview.py -f <sample> -O 1 --ollama-model <model>
```

**Anthropic (only if user explicitly requests AND PAP:GREEN or WHITE):**
```bash
export ANTHROPIC_API_KEY=<key>
malwoverview.py -f <sample> -A 1
```

> If neither available, interpret malwoverview output directly without API enrichment.

### Step 5: PAP Reclassification
After analysis, evaluate findings and recommend PAP adjustment if warranted.

**Escalate (tighten PAP) if:**
- Active C2 infrastructure or live campaign IOCs found
- Nation-state or APT attribution indicators
- Targeted attack artifacts (org-specific strings, internal hostnames)
- Zero-day or unpublished exploit code

**De-escalate (loosen PAP) if:**
- Known clean file / FP confirmed by hash
- Commodity malware already publicly documented
- No novel IOCs

Always present as recommendation — user confirms before PAP changes:

> 💡 **PAP Reclassification Suggestion**
> Current: PAP:[LEVEL] → Recommended: PAP:[NEW LEVEL]
> Reason: [specific finding]
> Confirm? (yes / no / keep current)

---

## Output Format

Always produce a markdown report with this structure:

```markdown
# Malware Analysis Report

## Sample Info
| Field | Value |
|---|---|
| Filename | |
| MD5 | |
| SHA256 | |
| File Type | |
| Size | |
| PAP Classification | |

## Static Analysis

### Strings of Interest
- Notable strings (URLs, IPs, registry keys, commands...)

### PE / Structure Analysis
- Imports, exports, sections, packing indicators

### Entropy
- Overall entropy, packed/encrypted section indicators

### YARA Matches
- Rule name, description

## IOCs
| Type | Value | Notes |
|---|---|---|
| Hash | | |
| URL | | |
| IP | | |
| Domain | | |
| Registry | | |
| File path | | |

## AI Enrichment
Summary from Anthropic/Ollama interpretation.

## Verdict
**Classification:** [Clean / Suspicious / Malicious / Unknown]
**Confidence:** [Low / Medium / High]
**Summary:** Brief description of likely behavior and purpose.

## Recommended Next Steps
- What to investigate further
- Whether dynamic analysis is warranted (note PAP constraints)
```

---

## PAP Constraints Summary

| PAP Level | VT Lookup | External API | Local Analysis |
|---|---|---|---|
| WHITE | ✅ | ✅ | ✅ |
| GREEN | ✅ | ✅ community | ✅ |
| AMBER | ❌ | ❌ | ✅ |
| AMBER+STRICT | ❌ | ❌ | ✅ local only |
| RED | ❌ | ❌ | ✅ isolated env |
