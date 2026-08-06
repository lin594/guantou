# API v1 Domain Design

## Requirements

- Preserve the validated `Can / Nameplate / Flavor / Package / Shelf` product boundaries.
- Replace the ambiguous `FlavorVariant` with a pronunciation record scoped by package, flavor, and dialect.
- Model only dialect relationships; create local-variety nodes when evidence requires finer granularity.
- Keep the API contract stable and independent from PR or issue progress.
- Provide a machine-readable contract and cross-referenced human design documentation.

## Design

```text
Package ─┐
         ├─ Pronunciation ─ Dialect
Flavor  ─┘         │
                   └─ Can[]

Can ─ Nameplate(package_id, flavor_id)
```

`Dialect` is a lazily populated tree with stable IDs and DNS-like qualified codes. `Pronunciation(package_id, flavor_id, dialect_id)` is a normalized dictionary statement; `Can` is recording evidence; `Nameplate` remains a user claim.

The normative API is stored in `docs/api/v1/openapi.yaml`. Human semantics live in `docs/api/v1/README.md`; the long-lived decision and alternatives are recorded in ADR-0001.

## Key decisions

- Keep root API paths for v1.
- Use Bearer authentication and `{code, message, data, request_id}` errors.
- Keep Package and Flavor as separate reusable resources.
- Preserve `FlavorPackage.mapping_type` through structured `package_links`; Pronunciation still stores the three direct foreign keys.
- Use explicit subtree filtering instead of implicit parent-includes-all-descendants behavior.
- Do not store audio on Pronunciation; link one or more Can records as evidence.
- Do not duplicate the YAML contract as committed JSON.

## Risks

- Migrating legacy `FlavorVariant` rows must split normalized pronunciation metadata from audio evidence without losing provenance.
- Dialect reparenting changes qualified codes, so aliases must be retained.
- Pronunciation duplicate detection requires reviewer workflow rather than a simplistic three-column unique constraint.

## Acceptance examples

- The written form “行” can map to both “行走动作” and “行业类别”.
- “行” meaning “行走动作” can have different Pronunciation rows under 莆田片 and 仙游片.
- A local variety can be added below 仙游片 without pre-creating unrelated county or town nodes.
- Several Can recordings can support one Pronunciation without duplicating the normalized dictionary record.
