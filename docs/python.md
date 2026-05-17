# Python API

In-process bindings for SNOExtract. Best when you want zero IPC overhead and direct access to the entity objects from Python code.

The README Quickstart covers installation; this page focuses on what's worth knowing once it's installed.

## Extract entities

```python
from snoextract import Pipeline

pipeline = Pipeline.load("./data")
result = pipeline.process("Patient on Metformin 1g BD for diabetes mellitus.")

for e in result.entities:
    print(e.text, e.cui, e.name, e.semantic_type)
```

Output:

```
Metformin 372567009 Metformin (substance) substance
diabetes mellitus 73211009 Diabetes mellitus (disorder) disorder
```

Useful entity attributes:

| Attribute | Type | Notes |
|---|---|---|
| `text`          | `str` | Matched surface form from the input |
| `start`, `end`  | `int` | Character offsets (half-open: `text[start:end]`) |
| `cui`           | `str` | SNOMED CT concept ID (decimal string) |
| `name`          | `str` | Canonical SNOMED preferred term |
| `semantic_type` | `str` | `disorder`, `finding`, `substance`, `procedure`, `body structure`, … |
| `similarity`    | `float` | Linker match score, 0.0–1.0 |
| `context`       | `ContextFlags` | Negation / uncertainty / historicity — see below |
| `kind`          | `EntityKind`   | Discriminator (`"medication"`, `"disorder"`, `"vital"`, `"lab"`, `"concept"`) — unlocks `med_info` and `value` |
| `section`       | `str` or `None` | Section the entity appeared in, if section detection found one |

`Pipeline.load` is the one-shot loader; for repeated runs in long-lived processes, keep the `Pipeline` instance and call `.process()` per note — loading is the slow part (~100 ms), inference is fast.

## Context flags

Every entity carries a `ContextFlags` object on `entity.context` that records how the surrounding text qualifies the concept:

```python
text = "Patient denies chest pain. History of asthma. Possibly UTI."
for e in pipeline.process(text).entities:
    c = e.context
    print(f"{e.text:12s} neg={c.is_negated}  unc={c.is_uncertain}  hist={c.is_historical}  cert={c.certainty!r}")
```

Output:

```
chest pain   neg=True   unc=False  hist=False  cert='negated'
asthma       neg=False  unc=False  hist=True   cert='affirmed'
UTI          neg=False  unc=True   hist=False  cert='possible'
```

Available flags:

| Attribute | Type | Meaning |
|---|---|---|
| `is_negated`    | `bool` | Concept is explicitly denied (*"no fever"*, *"denies chest pain"*) |
| `is_uncertain`  | `bool` | Hedged or possible (*"possibly UTI"*, *"query asthma"*) |
| `is_historical` | `bool` | Past, not current (*"history of stroke"*, *"previous MI"*) |
| `is_affirmed`   | `bool` | Asserted as present (the default for unmarked mentions) |
| `is_resolved`   | `bool` | Past *and* explicitly resolved (*"asthma resolved"*) |
| `certainty`     | `str`  | Summary label: `"affirmed"`, `"negated"`, `"possible"` |
| `experiencer`   | `str`  | Who the concept refers to: `"patient"`, `"family"`, `"other"` |

For batch filtering, the booleans are the simplest path:

```python
present = [e for e in result.entities if not e.context.is_negated and not e.context.is_historical]
```

…or use `certainty` if you want a single field to switch on.

## Medication details

When `entity.kind.tag == "medication"`, the pipeline also parses dose, frequency, and action from the surrounding text:

```python
text = "Started Metformin 1g BD and Atorvastatin 40mg nocte."
for e in pipeline.process(text).entities:
    if e.kind.tag == "medication":
        m = e.kind.med_info
        print(f"{e.text:14s} dose={m.dose!r:7s} freq={m.frequency!r:8s} action={m.action!r}")
```

Output:

```
Metformin      dose='1g'    freq='BD'      action='started'
Atorvastatin   dose='40mg'  freq='nocte'   action='started'
```

`MedInfo` attributes:

| Attribute | Type | Notes |
|---|---|---|
| `dose`        | `str` or `None` | e.g. `"1g"`, `"40mg"`, `"500 mcg"` |
| `frequency`   | `str` or `None` | e.g. `"BD"`, `"nocte"`, `"QID PRN"` |
| `action`      | `str` or `None` | e.g. `"started"`, `"ceased"`, `"increased"` |
| `full_start`, `full_end` | `int` | Character offsets covering medication + modifiers |
| `full_text`   | `str` | The medication phrase as it appeared in the note (e.g. `"Metformin 1g BD"`) |

Useful for medication reconciliation, dose-change detection, and bulk extraction of structured prescribing data.

## Sections

If the input has section headers (HPI, PMHx, Medications, Assessment, …), the pipeline detects them and tags each entity with the section it appeared in:

```python
note = """HPI: Patient presents with chest pain.
PMHx: Hypertension, diabetes mellitus.
Medications: Metformin 1g BD.
Assessment: Likely angina."""

result = pipeline.process(note)

for s in result.sections:
    print(f"{s.category!r:32s} header={s.header.text!r}")

for e in result.entities:
    print(f"  {e.text!r:20s} section={e.section!r}")
```

Output:

```
'history_of_present_illness'     header='HPI:'
'past_medical_history'           header='PMHx:'
'medications'                    header='Medications:'
'assessment'                     header='Assessment:'
  'chest pain'           section='history_of_present_illness'
  'Hypertension'         section='past_medical_history'
  'diabetes mellitus'    section='past_medical_history'
  'Metformin'            section='medications'
  'angina'               section='assessment'
```

Lets you filter by clinical context — e.g. *"only diagnoses from Assessment"*, *"only medications listed under Medications"* — without writing your own section detector.

## Tuning the match threshold

`process()` accepts a per-call `min_similarity` (0.0–1.0) that overrides the pipeline default. Higher = fewer fuzzy matches, fewer false positives:

```python
for thr in (0.85, 0.95):
    r = pipeline.process("diabeties", min_similarity=thr)
    print(f"thr={thr}: {[(e.text, e.cui, round(e.similarity, 2)) for e in r.entities]}")
```

Output:

```
thr=0.85: [('diabeties', '73211009', 0.89)]
thr=0.95: []
```

The default lives in `pipeline_config.toml`; the per-call override is the lightweight way to tighten or loosen for one call without restarting.
