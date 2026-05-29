# Contributing to morphkit-dictionary

This repo accepts two categories of contributions: **new morphs / loci** and **corrections to existing entries**. Both follow the same PR process.

Before opening a PR, validate your change locally:

```bash
npx ajv-cli validate -s schema.json -d dictionary.json
```

---

## What belongs here vs. the engine repo

| Contribution | Repo |
|---|---|
| New morph, locus, allele, combo, or lethality entry | **This repo** |
| Fix an allele name, inheritance type, or defect label | **This repo** |
| Engine logic, probability math, aggregator, worker, types | [trailofdad/morphkit](https://github.com/trailofdad/morphkit) |

---

## Adding a new locus

1. Add an entry under `loci` in `dictionary.json`. The key is the `locusId` (snake_case).
2. Set `id` equal to the map key.
3. Choose the correct `inheritance` value (see the README schema table).
4. Set `isSexLinked: true` only if the allele is transmitted via the sex chromosomes (e.g. Banana).
5. Always include a `"normal"` allele.
6. Run the validator. Fix any schema errors before opening a PR.

```jsonc
// example: adding a new recessive locus
"ultramel_locus": {
  "id": "ultramel_locus",
  "name": "Ultramel",
  "inheritance": "recessive",
  "isSexLinked": false,
  "alleles": {
    "normal":   { "id": "normal",   "name": "Normal"   },
    "ultramel": { "id": "ultramel", "name": "Ultramel" }
  }
}
```

---

## Adding a combo

Register a `ComboDefinition` only when the market name is well-established and differs from what the engine would produce by concatenating the component phenotype names. "Pastel Clown" does not need a combo entry; "Freeway" does.

```jsonc
{
  "marketName": "Freeway",
  "requiredGenotype": {
    "yellowbelly_complex": ["yellowbelly", "asphalt"]
  }
}
```

Allele IDs in `requiredGenotype` must exactly match the IDs in the referenced locus's `alleles` map.

---

## Adding a lethality rule

Add a `LethalComboDefinition` when a homozygous (or specific heterozygous) genotype is not viable. Do not remove the genotype from the engine's output — setting `isLethal: true` is how UIs warn breeders.

```jsonc
{
  "triggerGenotype": {
    "spider_complex": ["spider", "spider"]
  }
}
```

---

## Adding an allele defect

Set `defects` on an allele when it carries a congenital or neurological risk in the heterozygous state. The engine propagates these into `congenitalWarnings` on every outcome that carries the allele.

```jsonc
"spider": {
  "id": "spider",
  "name": "Spider",
  "defects": ["Neurological Wobble"]
}
```

---

## Naming conventions

| Item | Convention | Example |
|---|---|---|
| `locusId` / map key | `snake_case` | `cinnamon_complex` |
| `alleleId` / map key | `snake_case` | `black_pastel` |
| `name` (locus) | Title Case; append `(Line Name)` for multi-line recessives | `Axanthic (VPI Line)` |
| `name` (allele) | Title Case; use ` / ` to join synonyms | `Banana / Coral Glow` |
| `marketName` (combo) | Market-standard Title Case | `Freeway`, `Super Pastel` |

---

## Evidence requirements

New locus entries should be based on publicly documented genetic work or well-established community consensus. For disputed morphs (e.g. different axanthic lines that may or may not be allelic), open an issue to discuss before submitting a PR. Include a brief note in the PR description citing the source or community consensus behind the entry.

---

## Versioning

Bump `version` in `dictionary.json` using SemVer. The release workflow reads this field and creates a matching immutable git tag (`v{version}`) automatically on merge.

- **Patch** (`1.0.x`): correcting a name, adding a missing defect label, fixing a typo.
- **Minor** (`1.x.0`): adding new loci, alleles, combos, or lethality rules.
- **Major** (`x.0.0`): removing or renaming a locus/allele ID that engine consumers reference by key — this is a breaking change for any SPA pinned to the old version.

Always update `lastUpdated` to the current UTC timestamp alongside the version bump.

If your change does not affect the data (e.g. a README or workflow edit), do not bump the version — the existing tag already covers those commits and no new CDN URL is needed.

---

## How releases work

Every merge to `main` triggers `.github/workflows/release.yml`, which:

1. Validates `dictionary.json` against `schema.json` — **if validation fails, the release is blocked**.
2. Reads `version` from `dictionary.json`.
3. Creates a GitHub Release tagged `v{version}` **only if that tag does not already exist**.

The release notes include the exact immutable CDN URL for that version:
```
https://cdn.jsdelivr.net/gh/trailofdad/morphkit-dictionary@{version}/dictionary.json
```

SPA maintainers update their `DICTIONARY_VERSION` environment variable to consume the new release.

---

## PR checklist

- [ ] `npx ajv-cli validate -s schema.json -d dictionary.json` passes locally
- [ ] `version` is bumped (patch / minor / major as appropriate) and `lastUpdated` is updated
- [ ] `locusId`, `alleleId`, and `name` follow the naming conventions above
- [ ] New loci include a `"normal"` allele
- [ ] Allele IDs in `requiredGenotype` / `triggerGenotype` match the corresponding locus `alleles` map
- [ ] PR description includes a brief source or consensus note for new morphs
