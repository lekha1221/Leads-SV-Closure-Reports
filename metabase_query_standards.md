# Metabase query standards (Inn circles / CRM parity)

## Root cause (Pre SV vs Inn circles “scheduled”)

`stage = 'Pre Site Visit'` alone is **not** a schedule count. July Pre SV reasons include personal call, post call, missed call, automated follow-up, etc.

Inn circles **scheduled** = history rows whose **`reason`** text is a real site-visit schedule (physical or virtual), e.g.:

| reason | meaning |
|--------|---------|
| `site visit scheduled` | normal / physical |
| `virtual site visit scheduled` | virtual |
| `site visit scheduled by ai-core` | AI physical |
| `virtual site visit scheduled by ai-core` | AI virtual |

**Athena check (July 2026, AI-driven schedules):**  
`reason IN (... by ai-core)` → **60** leads / **18** Pre-launch Web.  
Prior Metabase (current AI assignee, any Pre SV reason) → **39** / **11**.  
Inn circles target cited → **43** / **19** (same family; remaining gap is usually UI project/date filters).

Do **not** use born-leads “exclude AI pools + 4 projects” for this AI schedule card — that universe is hundreds/thousands.

---

## Locked conventions

| Rule | Implementation |
|------|----------------|
| Grain | `COUNT(DISTINCT CAST(l.leadid AS varchar))` |
| Source | `leadsources` via **`l.bornsource`** |
| Project | `projects.name` (optional Metabase filter) |
| Date | `leadstagehistories.createdat_ist` calendar window |
| Schedule event | `stage = 'Pre Site Visit'` **and** reason matches schedule text below |
| AI schedule card | Prefer **reason** `*by ai-core*` / `*by ai*` — **not** current `assignedto` pool include/exclude |

### Reason filter (virtual **or** normal)

```sql
AND LOWER(CAST(h.reason AS varchar)) LIKE '%schedul%'
AND (
  LOWER(CAST(h.reason AS varchar)) LIKE '%site visit%'
  OR LOWER(CAST(h.reason AS varchar)) LIKE '%virtual%'
)
```

### AI-driven schedules (Inn circles AI card)

```sql
AND (
  LOWER(CAST(h.reason AS varchar)) LIKE '%by ai%'
  OR LOWER(CAST(h.reason AS varchar)) LIKE '%ai-core%'
  OR LOWER(CAST(h.reason AS varchar)) LIKE '%ai core%'
)
```

---

## Schedule / Pre Site Visit — Inn circles AI (paste-ready)

```sql
WITH pre_sv AS (
  SELECT
    CAST(h.lead AS varchar) AS lead_oid,
    MIN(h.createdat_ist) AS pre_sv_at
  FROM "leadsdb-prod"."leadstagehistories" h
  WHERE TRIM(CAST(h.stage AS varchar)) = 'Pre Site Visit'
    AND h.createdat_ist >= CAST({{start_date}} AS timestamp)
    AND h.createdat_ist < date_add('day', 1, CAST({{end_date}} AS timestamp))
    /* virtual or normal site-visit schedule */
    AND LOWER(CAST(h.reason AS varchar)) LIKE '%schedul%'
    AND (
      LOWER(CAST(h.reason AS varchar)) LIKE '%site visit%'
      OR LOWER(CAST(h.reason AS varchar)) LIKE '%virtual%'
    )
    /* Inn circles AI schedule card */
    AND (
      LOWER(CAST(h.reason AS varchar)) LIKE '%by ai%'
      OR LOWER(CAST(h.reason AS varchar)) LIKE '%ai-core%'
      OR LOWER(CAST(h.reason AS varchar)) LIKE '%ai core%'
    )
  GROUP BY 1
)

SELECT
  date_format(p.pre_sv_at, '%Y-%m') AS month_key,
  date_format(p.pre_sv_at, '%M %Y') AS month_name,
  COALESCE(NULLIF(TRIM(CAST(ls.name AS varchar)), ''), 'UNKNOWN') AS "Source",
  COALESCE(NULLIF(TRIM(CAST(pr.name AS varchar)), ''), 'UNKNOWN') AS "Project",
  COUNT(DISTINCT CAST(l.leadid AS varchar)) AS schedule_leads
FROM pre_sv p
JOIN "leadsdb-prod"."leads" l
  ON CAST(l._id AS varchar) = p.lead_oid
LEFT JOIN "leadsdb-prod"."leadsources" ls
  ON CAST(ls._id AS varchar) = CAST(l.bornsource AS varchar)
LEFT JOIN "leadsdb-prod"."projects" pr
  ON CAST(pr._id AS varchar) = TRIM(CAST(l.project AS varchar))
WHERE l.leadid IS NOT NULL
  [[AND UPPER(TRIM(CAST(pr.name AS varchar))) = UPPER(TRIM({{project}}))]]
  [[AND COALESCE(NULLIF(TRIM(CAST(ls.name AS varchar)), ''), 'UNKNOWN')
        = TRIM({{source}})]]
GROUP BY 1, 2, 3, 4
ORDER BY month_key, "Source", "Project";
```

**July 2026 self-check (Athena):** total ≈ **60**, Pre-launch Web ≈ **18**.

---

## All pools — scheduled only (human + AI)

Same reason filter **without** the `by ai` clause. Optionally exclude current AI pools if matching born-leads ops cards:

```sql
-- optional: drop current AI / AI-CORE / AI CORE assignees
AND ac.lead_oid IS NULL
```

---

## Born leads (different card)

Exclude pool **`AI-CORE` / `AI CORE` only**, keep **`AI`**. Calendar IST months. Four projects SPECTRA/LOFT/LANDMARK/BROADWAY. **No** schedule-reason filter on born volume.
