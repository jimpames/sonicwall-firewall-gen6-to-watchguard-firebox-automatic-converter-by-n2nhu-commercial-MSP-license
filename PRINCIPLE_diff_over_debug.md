# The $1 / $100 Principle

> "Every dollar we spend on diff is maybe a hundred dollars we don't spend on debug."
> — Captain, n2nhu lab, 3 Jun 26

This is the principle that organizes the toolkit's validator strategy. It's worth writing down because the math is easy to forget when an immediate bug is staring at you.

---

## The Two Oracles

Every WatchGuard schema invariant — every rule the importer enforces — can be checked by one of two oracles:

| Oracle | When it runs | Cost per check | Cost per failed check |
|---|---|---|---|
| **wg_validator** (the $1 oracle) | Locally, before push | Milliseconds | Dev fixes locally, re-runs |
| **Firebox importer** (the $100 oracle) | After file push | 30-60s import + 60s reload | Lab time, brick risk, customer downtime, reverse-engineering opaque error messages |

The Firebox is authoritative — it's what we ultimately have to satisfy. But it's an *expensive* oracle. Every round-trip costs real time and carries real risk (especially when the import partially succeeds before failing, leaving the device in an indeterminate state).

The validator is *cheap*. Running it costs nothing. Adding a new rule to it costs minutes. Once a rule is encoded, that class of bug never reaches the Firebox again — from any operator, on any future customer migration.

## The Asymmetry

When the Firebox catches a bug, it tells you:
- One line number
- One terse error message
- No path forward — the operator has to figure out what to fix and how

When the validator catches the same bug, it tells you:
- The xpath of the offending element
- The specific rule violated
- The pattern that should be there instead (grounded in jpa.xml)
- Often the exact INI directive to change

**Same bug, vastly different debugging cost.** And the validator catches it before any hardware is touched, before any customer downtime is risked.

## The Compounding Property

Each Firebox 400 error during testing should produce, at minimum:
1. A fix in the emit config or value map (resolves the immediate bug)
2. A new rule in the validator (catches the class of bug forever)

Step 2 is the asset. The validator becomes a permanent corpus of grounded WatchGuard schema invariants. Future migrations — new customers, new SonicWall configs, new operators — all benefit from rules added today.

**A $1 investment today saves $100 every future round.**

## What This Means in Practice

When you face a choice between:

- **(A)** "Fix the immediate bug and ship the file" (cheap now, no compounding)
- **(B)** "Fix the bug AND encode the rule in the validator" (slightly more work now, compounds forever)

Always choose (B). The extra time on the validator rule pays back the first time it fires for any future config.

When you face a choice between:

- **(A)** "Skip the validator with `--no-validate-output` to get a file faster"
- **(B)** "Fix the validator's blind spot, then emit"

Always choose (B). Skipping the validator means trading $1 of local work for $100 of lab debugging. That's the inverse of where you want the leverage.

## The Phase 9 Audit

Bugs caught by the Firebox during the 2 Jun 26 lab T-30 smoke test, and what was added to the validator in response:

| Firebox error | Validator rule added |
|---|---|
| `host-ip-addr not expected, Expected (type)` | `POLYMORPHIC_RULES`: `<member>` requires `<type>` |
| `protocol not expected, Expected (type)` | Same rule, ordering check (`@first`) |
| `member: Missing child element(s)` | `_check_member_payloads`: `<member type=1>` requires payload |
| `'echo-reply' not Port-Type` | Tightened `<server-port>` to `value_type=int` |
| `'IPCOMP' not unsignedShort` | `PATH_AWARE_VALUE_TYPES`: `<protocol>` in service-member must be int |

Each one of these rules will fire next time. None of them will reach the Firebox again. **That's the principle working as designed.**

## The Bigger Frame

The schema-learner subsystem already understands this: build a corpus of approved patterns, refer to it instead of re-deriving from scratch each time. The validator IS the same idea applied to one layer up — a corpus of *negative* patterns (things that look right but the Firebox rejects).

Together they form the toolkit's two-sided knowledge base:
- **Schema-learner**: "what's the right shape for X?" (approved corpus)
- **Validator**: "what mistakes look superficially like X but aren't?" (banned/grounded corpus)

Both compound. Both encode senior-engineer judgment into infrastructure. Both make the next migration cheaper than the last.

That's the toolkit. That's the bet. And Captain's $1/$100 principle is the math underneath.

---

*Written 3 Jun 26 in response to Captain's exact words during the lab T-30 smoke test. Drop this in front of anyone who asks why the validator gets hardened every time we hit a new Firebox error.*
