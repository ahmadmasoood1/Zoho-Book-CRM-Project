=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-014
DATE: 2026-10-07

REASON:          The seed loaded every known intent (incl. M11) for completeness. report.schedule
                 having needs_pin=false was a mistake, not a decision. Also, check_permission()
                 ignores intents.active, so an intent for a later milestone would be allowed as
                 soon as it is seeded.

RECOMMENDATION:  Change:
  1. Keep seeding future intents, but with active = false.
  2. Each milestone activates its own intents in its first migration:
       update intents set active = true where milestone = 'M<n>';
     This is explicit and traceable in Git.
  3. check_permission(): join intents and require i.active = true. Inactive = default deny.
     WF-04 builds its catalogue from active intents only, so the LLM never offers an unbuilt
     feature.
  4. Seed §7.2: add report.schedule to the needs_pin list (same as report.run). Staff stays
     not allowed.
  5. Architecture §8: list report.schedule as a Should (M11) intent.

IMPACT:
  - Database.md §5.4 (check_permission) and §7.1/§7.2 (seed); Architecture §8.
  - One activation line per milestone migration.
  - Tests (M2):
    (a) an inactive intent → PERMISSION_DENIED for the Owner too;
    (b) report.schedule needs_pin = true;
    (c) M2 intents (user.*, audit.summary) active after the M2 migration.
  - Very low effort; blocks M2.
