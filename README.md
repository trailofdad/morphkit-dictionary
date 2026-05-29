# morphkit-dictionary

The official trait dictionary for the [morphkit](https://github.com/trailofdad/morphkit) genetic calculation engine. This repository contains all locus definitions, allele registrations, named combo mappings, lethality rules, and polygenic tags for ball python morphs.

The dictionary is versioned and distributed independently from the engine so that morph data can be updated without shipping a new library release.

## CDN

### Development

During local development, `@latest` is convenient:

```ts
import { syncDictionary } from 'morphkit';

const dictionary = await syncDictionary(
  'https://cdn.jsdelivr.net/gh/trailofdad/morphkit-dictionary@latest/dictionary.json'
);
```

**Do not use `@latest` in production.** jsDelivr caches the `@latest` pointer at edge nodes for up to 24 hours. Combined with morphkit's own stale-while-revalidate browser cache, a critical dictionary fix could take two full invalidation cycles — up to 48 hours — to reach a user after it is merged.

### Production

Target a specific version tag. jsDelivr permanently caches immutable version URLs, so there is no propagation delay and no stale-edge risk:

```
https://cdn.jsdelivr.net/gh/trailofdad/morphkit-dictionary@1.0.0/dictionary.json
```

**Recommended pattern** — drive the version from an environment variable so a dictionary update requires only a config change, not a code deploy:

```ts
const version = process.env.DICTIONARY_VERSION ?? '1.0.0';
const dictionary = await syncDictionary(
  `https://cdn.jsdelivr.net/gh/trailofdad/morphkit-dictionary@${version}/dictionary.json`
);
```

When a new dictionary version is released, update `DICTIONARY_VERSION` in your environment and redeploy (or push it from a config API without a full redeploy). The release notes for each version include the exact CDN URL to use.

### Automated releases

Every merge to `main` runs the [release workflow](./.github/workflows/release.yml):

1. Validates `dictionary.json` against `schema.json` — the push is blocked if validation fails.
2. Reads `version` from `dictionary.json`.
3. If that version tag does not already exist on the repo, creates a GitHub Release and tags the commit `v{version}`.

This means the immutable CDN URL for a release is available seconds after a PR merges — not 24 hours later. To publish a new version, bump `version` (and `lastUpdated`) in `dictionary.json` as part of your PR.

A stale-while-revalidate cache (24-hour TTL in `localStorage`) means most page loads resolve instantly from cache regardless of which URL pattern you use.

## Dictionary schema

All data lives in a single file: [`dictionary.json`](./dictionary.json). The JSON Schema is at [`schema.json`](./schema.json). Validate locally with [ajv-cli](https://www.npmjs.com/package/ajv-cli):

```bash
npx ajv-cli validate -s schema.json -d dictionary.json
```

### Top-level fields

| Field | Type | Description |
|---|---|---|
| `version` | `string` | SemVer string, e.g. `"1.0.0"` |
| `lastUpdated` | `string` | ISO 8601 timestamp of the last change |
| `loci` | `Record<string, LocusDefinition>` | All registered loci, keyed by `locusId` |
| `combos` | `ComboDefinition[]` | Named market combos (e.g. Freeway, Pewter) |
| `lethalCombos` | `LethalComboDefinition[]` | Genotypes flagged as embryonically lethal |
| `polygenicTags` | `string[]` | Valid polygenic trait labels |

---

### `LocusDefinition`

```jsonc
{
  "id": "clown_locus",           // snake_case, must match its map key
  "name": "Clown",               // human-readable display name
  "inheritance": "recessive",    // see InheritanceType below
  "isSexLinked": false,
  "alleles": {
    "normal": { "id": "normal", "name": "Normal" },
    "clown":  { "id": "clown",  "name": "Clown"  }
  }
}
```

**`InheritanceType`** — one of:

| Value | Meaning |
|---|---|
| `"recessive"` | Two copies required for phenotype expression |
| `"dominant"` | One copy always expresses (but may still be lethal when homozygous — register in `lethalCombos`) |
| `"incomplete_dominant"` | One copy expresses; two copies produce a distinct "super" phenotype; also used for allelic complexes |
| `"polygenic"` | Multi-gene trait; not modeled by the Punnett engine |

Every locus **must** include a `"normal"` allele. Allelic complexes (e.g. Yellowbelly Complex, Cinnamon Complex) are modeled by adding more than two alleles to a single locus.

---

### `AlleleDefinition`

```jsonc
{
  "id": "spider",                          // snake_case, must match its map key
  "name": "Spider",                        // human-readable display name
  "defects": ["Neurological Wobble"]       // optional; congenital / neurological warnings
}
```

`defects` is set on alleles that carry a congenital or neurological risk even in the heterozygous state. The engine copies these into `congenitalWarnings` on the `AggregatedOutcome` so your UI can surface them.

---

### `ComboDefinition`

Register a named combo when a specific allele pairing produces a morph with a market name distinct from the component names.

```jsonc
{
  "marketName": "Freeway",
  "requiredGenotype": {
    "yellowbelly_complex": ["yellowbelly", "asphalt"]
  }
}
```

All listed locus conditions must match for the combo to trigger. Combos involving multiple loci are supported by adding more entries to `requiredGenotype`.

**When to register a combo:** only when the market name is well-established and the pairing is unambiguous. Multi-locus combos like "Pastel Clown" do not need entries — the engine concatenates unregistered phenotype names automatically.

---

### `LethalComboDefinition`

```jsonc
{
  "triggerGenotype": {
    "spider_complex": ["spider", "spider"]
  }
}
```

When all listed locus conditions match, the engine sets `isLethal: true` on the `AggregatedOutcome`. The genotype is still returned in the output so UIs can warn the breeder rather than silently drop the outcome.

---

### `polygenicTags`

A flat list of valid polygenic trait labels (e.g. `"Jungle"`, `"Black Back"`). These are passed through from both parents to every offspring outcome without affecting probability math.

---

## Current morph coverage

### Recessive

| Locus ID | Name |
|---|---|
| `clown_locus` | Clown |
| `pied_locus` | Piebald |
| `albino_locus` | Albino |
| `axanthic_vpi_locus` | Axanthic (VPI Line) |
| `lavender_albino_locus` | Lavender Albino |
| `genetic_stripe_locus` | Genetic Stripe |
| `desert_ghost_locus` | Desert Ghost |

### Incomplete dominant

| Locus ID | Name | Notes |
|---|---|---|
| `pastel_locus` | Pastel | Super Pastel registered |
| `enchi_locus` | Enchi | Super Enchi registered |
| `ghi_locus` | GHI | Super GHI registered |
| `spider_complex` | Spider | Homozygous lethal; Neurological Wobble defect on `spider` allele |
| `cinnamon_complex` | Cinnamon Complex | Allelic: Normal / Cinnamon / Black Pastel; Pewter, Super Cinnamon, Super Black Pastel registered |
| `yellowbelly_complex` | Yellowbelly Complex | Allelic: Normal / Yellowbelly / Asphalt; Ivory, Freeway registered |

### Sex-linked

| Locus ID | Name | Notes |
|---|---|---|
| `banana_locus` | Banana / Coral Glow | `isSexLinked: true`; engine maps to correct sex from sire's heterogametic passing |

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the full process.
