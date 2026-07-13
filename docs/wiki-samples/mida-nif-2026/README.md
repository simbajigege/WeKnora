# MIDA NIF Manufacturing Guidelines 2026 Wiki Sample

This directory is an offline sample of the WeKnora LLM Wiki output shape for:

- Source PDF: `dataset/samples/Malaysia_MIDA_NIF_Manufacturing_Guidelines_2026.pdf`
- Document title: `Guidelines of Tax Incentives for New Investment in the Manufacturing Sector under the New Incentive Framework (NIF)`
- PDF date shown in document: `As at 15.01.2026`
- Pages: 19

The real product path is asynchronous:

```text
PDF -> DocReader Markdown -> chunks -> wiki:ingest
    -> summary/<knowledge_id>
    -> entity/* and concept/*
    -> index/folders/link finalization
```

This sample was generated offline from extracted PDF text, so `chunk_refs` in `manifest.json` are stable illustrative IDs rather than database UUIDs.

## Sample Pages

- `summary/malaysia-mida-nif-manufacturing-guidelines-2026.md`
- `entity/malaysian-investment-development-authority.md`
- `entity/new-incentive-framework.md`
- `entity/national-investment-aspirations.md`
- `entity/new-industrial-master-plan-2030.md`
- `concept/special-tax-rate.md`
- `concept/investment-tax-allowance.md`
- `concept/nia-scorecard.md`
- `concept/tiering-approach.md`
- `concept/global-minimum-tax.md`
