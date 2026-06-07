# Why We Emit XML, Not CLI

> **A standalone paper on a deliberate architectural choice.**
>
> SonicWall → WatchGuard Migration Toolkit
> n2nhu lab • Captain Jim Ames + Claude
> 2026-05-13

---

## Abstract

The n2nhu lab migration toolkit emits **WatchGuard XML configuration files** as its translation target, not WatchGuard Fireware CLI commands. This is a deliberate architectural choice, not an oversight or a "we'll get to it later" deferment. This paper documents the four reasons why XML is the correct target and CLI-as-output is not viable for production migrations — not now, and not in the foreseeable architecture of the toolkit.

The argument is not "the CLI is bad." The Fireware CLI is a competent, well-documented interface for operators working interactively with one device at a time. The argument is that **CLI is the wrong shape of interface for bulk, atomic, declarative migration work**, and that XML import is the right shape because it matches the problem.

The four reasons, briefly:

1. **CLI is imperative; migration is declarative.** XML expresses a target state; CLI expresses a sequence of state transitions. Translating to a sequence introduces ordering, dependency, and partial-failure concerns that translating to a state document does not.
2. **CLI cannot atomically replace a configuration.** Per the v2026.1.2/v12.11.8 reference, the CLI has no "import full configuration" command. Bulk replacement via CLI requires hundreds-to-thousands of individual commands. XML import is one transaction.
3. **CLI session state is the wrong cursor.** CLI sessions are interactive, stateful, lockable, and disconnect-prone. XML import is filesystem-mediated, non-interactive, and survives interrupted operator sessions.
4. **CLI surface area is narrower and version-pinned.** Several configuration domains are XML-configurable but not CLI-configurable (or are CLI-configurable only via specialized sub-commands). The CLI reference is firmware-version-specific in ways the XML schema is more stable than.

Each claim below is anchored in either the WatchGuard Fireware CLI Reference v2026.1.2/v12.11.8 or in observable behavior of the toolkit's existing emit pipeline.

---

## 1. CLI is imperative; migration is declarative

A migration is fundamentally a **declarative problem**: the operator knows the target state — "the customer's Firebox should have these N firewall policies, these M VPN tunnels, these K NAT rules, configured exactly this way." The migration's success criterion is "the running config matches the target state."

XML matches this shape directly. An emitted `migrated.xml` IS the target state, byte-for-byte. The operator can review it, diff it against a reference, validate it against a schema. The Firebox imports it, and if the import succeeds, the running config IS the target state. **One artifact, one transition, one outcome.**

CLI is fundamentally an **imperative problem**: each command is a state transition relative to whatever state the Firebox is currently in. To translate a target state into CLI commands, the toolkit would have to:

- **Model the current state of the device.** A `policy add foo` command means something different depending on whether `foo` already exists, partially exists, or is referenced by other config. The toolkit would need to query the device, model what's there, then plan the right transitions.
- **Order the commands correctly.** WatchGuard objects have reference dependencies — a policy references a service, a service references aliases, a tunnel references a gateway, a gateway references peer addresses. A CLI sequence must create objects in dependency order. The XML import handles this by representing the dependency graph as a tree.
- **Handle deletions explicitly.** XML import replaces the config in one operation. CLI must explicitly `no` every leftover element from the previous config that isn't in the target. Per the CLI reference, `no` is documented as the negation form across most commands (`no fips enable`, `no global-setting device-admin enable`, etc.) — useful, but **the toolkit would have to enumerate everything to remove, which means modeling the source config in detail**.

This isn't a hypothetical concern. The toolkit already has a concept-map and emit layer that targets WatchGuard's vocabulary. If it targeted CLI instead, it would also need:

- A **device state interrogator** (parse `show` output, build a model of current device state)
- A **diff engine** (compare target state vs. current state to derive the minimal command sequence)
- A **dependency-ordering planner** (sequence commands so prerequisites exist before references)
- An **error-recovery protocol** (if command 47 of 200 fails, what state is the device in?)

**All four are non-trivial. None are needed for XML import.** WatchGuard's XML importer handles atomicity, ordering, and replacement internally. We get those properties for free.

---

## 2. CLI cannot atomically replace a configuration

This is the structural fact that makes everything else moot. Per the Fireware CLI Reference v2026.1.2/v12.11.8:

> **The CLI has no "import configuration XML" command.**

The CLI's `import` command, scoped from page 12 of the reference, is for specific narrow file types:

- **Mobile VPN with IPSec user configuration files** — `export muvpn group-name [client-type client] to (location)` and corresponding `import` (page 423)
- **Dynamic routing configuration** — `import route-config` (page 2275, used because `vtysh` direct edits are unsafe)
- **Support snapshots** — `export support to (location)`

What's conspicuously absent: **there is no `import xml` or `restore-config` command** that takes a full Firebox XML and applies it to the device. The CLI's view of "import" is for *files associated with specific subsystems*, not for full device configuration replacement.

This means a CLI-targeted toolkit would have to produce, for a typical customer migration:

- ~25 `policy add ...` commands (one per access rule)
- ~50 `service add ...` commands (where customer-defined services exist)
- ~5-10 `bovpn-tunnel` commands (one per site-to-site VPN), each requiring multi-step Phase 1 / Phase 2 / gateway / tunnel definition
- ~10-30 `nat-policy ...` commands
- Address-object, alias, schedule definitions — dozens to hundreds
- Plus the `no ...` commands to remove whatever was on the device previously

**A typical mid-size customer migration would be 500-2000 individual CLI commands** that must be issued in the correct order over an interactive (or scripted-interactive) session, with the device potentially in production state.

Compare to the XML path:

```
$ python3 run_pipeline.py --input nsa3600.txt
$ # → migrated.xml produced (one file)
$ # → operator reviews migrated.xml
$ # → operator imports migrated.xml via Policy Manager or Web UI (one action)
```

**One file, one transition.** The atomicity is a feature of WatchGuard's import path, and the toolkit takes it as a given.

---

## 3. CLI session state is the wrong cursor

The CLI reference documents (pages 13-15) the WatchGuard CLI's **five command modes**: Main, Configuration, Interface, Link Aggregation, and Policy. To make a configuration change, an operator (or a toolkit) must:

1. Connect via serial or TCP/IP
2. Authenticate
3. Enter the appropriate mode (e.g., `configure` to enter Configuration mode)
4. Optionally descend further (`interface eth0` to enter Interface mode for one specific interface)
5. Issue commands
6. Exit modes back up the hierarchy

Three architectural problems for migration use:

### 3.1 The configuration file lock

Per page 764-766 of the reference:

> *"If the Firebox has been configured to allow more than one user with Device Administrator credentials to connect at the same time, and a Device Administrator has unlocked the configuration file to make changes, you cannot make changes to the configuration file until that Device Administrator has either locked the configuration file again or has logged out."*

This means **CLI-based configuration changes are mutually exclusive across operator sessions.** A migration that takes 30 minutes of CLI commands locks the configuration file for that entire duration. Other administrators cannot make changes. A senior engineer doing parallel work on another part of the device is blocked.

XML import, by contrast, is a single-transaction event that occupies the device for seconds, not minutes.

### 3.2 Session-state-dependent error recovery

Per pages 679-712 of the reference, the CLI has **five distinct error classes**:

- *Unrecognized command* — command doesn't exist
- *Incomplete command* — required parameter missing
- *Execution error* — semantic mismatch (referenced entity doesn't exist, etc.)
- *Syntax error* — `% Invalid input detected at '^' marker`
- *Ambiguous command* — truncated command matches multiple possibilities

Each error halts the offending command, but **leaves the session in its current mode with whatever previous commands already executed still committed**. There is no documented transaction or rollback primitive in the CLI reference.

For a migration toolkit emitting a 500-command sequence, this means: if command 247 fails (because of a typo in the toolkit's emit code, or a firmware quirk, or a transient state mismatch), the device is now in a **half-migrated state** — 246 commands applied, 254 commands not applied. Recovery requires either:

- Manually reverting the 246 partial changes (operator must know exactly what was applied), or
- Restoring a pre-migration backup (provided one was taken), or
- Continuing forward with manual fixes (requires operator to debug live)

XML import has one failure mode: the import either succeeds (whole config applied) or fails with diagnostics (no change applied). The atomicity is what makes the migration reviewable and rollbackable in a single artifact.

### 3.3 Network interruption mid-migration

CLI sessions are **stateful TCP or serial connections**. A network blip, an operator laptop sleeping, a firewall in the path resetting, or the Firebox itself momentarily reloading a service can drop the session. The toolkit would have to detect the disconnect, re-authenticate, restore the mode hierarchy, and figure out where in the command sequence to resume.

XML import is filesystem-mediated. Once the XML is on the device's filesystem (or staged for import), the import operation is local to the device. No session, no resumption, no interruption modeling.

---

## 4. CLI surface area is narrower and version-pinned

This argument is the most empirical and the one most likely to evolve. As the toolkit matures, we should re-test it periodically. The claim, today, anchored in the Fireware CLI Reference v2026.1.2/v12.11.8:

### 4.1 Several XML-configurable surfaces are not (or not fully) CLI-configurable

The XML schema we work against (the corpus has `T30jpa1.xml`, 657KB and 773 element types) covers configuration domains that the CLI reference either does not document at all or documents only for specific narrow operations:

- **Application Control rules** — visible as XML structures; not visible as first-class CLI commands in the reference's TOC
- **Content Filtering with category-level granularity** — XML elements exist; CLI reference's content filter section is narrower
- **Certain advanced threat features** (APT Blocker, Botnet Detection, GAV/IPS configuration tuning) — XML elements present; CLI coverage selective
- **HTML/Web Access Portal customization** — XML-driven; CLI reference does not document portal XML editing

This list will be **wrong in detail over time** — WatchGuard adds CLI coverage in firmware updates. But the pattern is consistent: **XML is the comprehensive surface, CLI is a curated subset.** WatchGuard's own GUI tools (Policy Manager, Fireware Web UI) work primarily with XML-shaped configurations and use the XML import path themselves.

### 4.2 The CLI reference is firmware-version-pinned

The reference we cite is titled **"Fireware v2026.1.2/v12.11.8"**. Commands change between firmware versions:

- New features add new commands
- Deprecated features remove or relabel commands
- Syntax for existing commands occasionally changes

A toolkit that emits CLI commands targets a **specific firmware version**. Customers on older firmware would receive commands their device doesn't understand. Customers on newer firmware might receive commands that have been renamed.

XML schemas evolve too — but the WatchGuard XML schema has a published versioning convention and the importer is forgiving of unknown elements (it ignores them) and supplies defaults for missing elements. **A toolkit emitting XML aligned to a 5-year-old schema typically still imports cleanly**; a toolkit emitting CLI commands from a 5-year-old firmware reference often does not.

---

## 5. The operational argument: GUI parity

A consideration the architectural argument doesn't capture, but operators care about deeply: **WatchGuard's own primary configuration tools work with XML.**

- **Policy Manager** (the Windows app most senior engineers use) opens, edits, and saves XML files. Import to a device pushes XML. Export from a device pulls XML.
- **Fireware Web UI** (the browser-based config tool) operates on XML under the hood. Backup/restore is XML.
- **WatchGuard Cloud / Centralized Management** uses XML as the wire format for pushing configurations to managed Fireboxes.

A toolkit that emits XML produces output that an operator can:
- Open in Policy Manager and visually review
- Edit in Policy Manager if a tweak is needed before deployment
- Import via the same path the operator uses for any other config change
- Diff against a previously-known-good config using standard XML tools

A toolkit that emits CLI produces output that:
- Cannot be opened in Policy Manager
- Cannot be visually reviewed except by reading the command sequence
- Cannot be deployed via the operator's normal workflow
- Cannot be diffed against a Policy Manager export

**The operator's existing skills, tooling, and verification workflows are all XML-shaped.** Producing CLI output would force a parallel workflow that operators would have to learn and trust separately from the workflow they already use. Producing XML output integrates with what they already know.

---

## 6. The brick-risk argument (and why it cuts both ways)

A claim worth examining honestly: it might seem that CLI is *safer* than XML import because the operator can issue commands one at a time and stop if something looks wrong. This deserves direct engagement.

The argument for CLI-as-safer:
- Each command can be reviewed before issued
- Partial application means partial blast radius
- The operator stays in the loop throughout

The argument that holds up under examination:

- **XML import is reviewable BEFORE commit.** The XML file is on disk before it's imported. The operator can open it in Policy Manager, review it, modify it, run it through the toolkit's validator, diff it against a reference — all without touching the device. The "review before commit" property is *stronger* for XML, not weaker.
- **CLI's partial-application property is a hazard, not a feature, during migration.** Migration is bulk replacement. A half-applied migration is a misconfigured firewall in production — potentially open ports where security policies were supposed to be, or closed ports where business traffic was supposed to flow. The all-or-nothing property of XML import is **safer** in this domain.
- **XML import does have its own brick risk.** A malformed XML that the device accepts can leave management interfaces unreachable, requiring a console-cable recovery. This is real. But it's the *same risk class* CLI would have if an operator typed a syntax-valid but semantically-disastrous `no policy management-https` command. The risk is in the *content* of the config, not the *delivery mechanism*.
- **The toolkit's principle binding addresses content risk regardless of delivery.** The schema-learner's no-claim-without-provenance gate, the triage workflow, the validator, the Phase 7 relevance filter — all of these check the *content* of the emitted config. They would apply equally if the emitter targeted CLI.

The brick-risk argument therefore reduces to: **XML import is reviewable before commit and atomic on commit. CLI is reviewable per-command but non-atomic on commit.** For migration, atomicity matters more than per-command review, because per-command review at scale is impractical — no senior engineer is going to eyeball 500-2000 CLI commands.

---

## 7. When CLI emission WOULD be the right call

Honest scoping: the toolkit shouldn't say "never CLI." There are workflows where CLI emission is the right choice, and we should know which ones so we don't apply the wrong tool when those come up:

| Use case | Right tool | Why |
|---|---|---|
| **Bulk migration of customer config** | XML | This paper. |
| **One-off interactive change** ("disable BOTNET temporarily on this customer's box") | CLI | One command. Imperative shape matches the task. |
| **Continuous-config-drift monitoring** | CLI's `show` commands | Read-only state interrogation. |
| **Emergency response** ("lock down management plane NOW") | CLI | Imperative urgency. Speed matters more than atomicity. |
| **Per-engineer ad-hoc tweaks** to a device already migrated by the toolkit | CLI | Surgical edits on a known baseline. |

The toolkit could, in principle, emit **CLI snippets as supplementary output** for specific narrow operations — for example, "after importing the XML, run these three CLI commands to enable a feature the XML import doesn't activate by default." This would be additive, not a replacement, and would target the narrow use cases where CLI is the right shape.

---

## 8. Conclusion

The decision to emit XML is not a preference — it is a structural consequence of the problem. Migration is:

- **Declarative** (target state, not transition sequence)
- **Atomic** (whole-config replacement, not incremental edits)
- **Auditable** (reviewable artifact before commit)
- **Interoperable** (the operator's existing tools speak XML)
- **Version-tolerant** (XML schema is more stable than CLI command syntax)

CLI is the wrong shape for all five properties. XML is the right shape for all five. The toolkit emits XML.

This is not closed-minded. If WatchGuard ships an `import-config-from-xml` CLI command in a future firmware, that's interesting — it would make CLI a *transport* for XML rather than a *substitute* for XML, and the argument about command shape would no longer apply. The toolkit's emit layer is configurable (`xml_emit_config.ini`) and could be supplemented with an output transport that wraps the XML in CLI delivery commands.

But that day is not today, per the v2026.1.2/v12.11.8 reference, and may never come. The toolkit's emit-XML architecture is correct for the foreseeable shape of WatchGuard's configuration model.

---

## Appendix A — Citation index

Every architectural claim in this paper is anchored in the WatchGuard Fireware Command Line Interface Reference v2026.1.2/v12.11.8:

| Claim | CLI Reference location |
|---|---|
| Five command modes (Main, Configuration, Interface, Link Aggregation, Policy) | TOC pp. 13-15 |
| Configuration mode hierarchy requires nested mode entry | p. 14 |
| Configuration file locking when multiple administrators connect | pp. 764-766 (also reiterated p. 4778) |
| Five error message categories (unrecognized, incomplete, execution, syntax, ambiguous) | pp. 679-712 |
| `import` command scope limited to specific file types (Mobile VPN profile, route-config, support snapshot) | pp. 423, 2275, 432 |
| `no` command form for negation/defaults | p. 1936 and throughout (e.g., p. 4777 `no global-setting device-admin enable`) |
| Import & Export via FTP/TFTP with full URL syntax | pp. 713-728 |
| No `import xml` or `restore-config` full-configuration command in the reference TOC | Established by search across full 11,663-line reference |

---

## Appendix B — What this paper deliberately does NOT argue

To keep the argument tight and defensible, the following claims are **not** made in this paper, even if they are sometimes made in other contexts:

- **"The CLI is bad."** It isn't. It's a competent operator tool for interactive work.
- **"The CLI is slow."** Speed is not the issue. Atomicity, declarative shape, and operator workflow fit are.
- **"WatchGuard wants us to use XML."** We don't know what WatchGuard wants. The argument is grounded in what's documented and what fits the migration problem.
- **"CLI emission would be impossible."** It would be possible but expensive — the toolkit would gain four substantial subsystems (state interrogator, diff engine, dependency planner, recovery protocol) for zero gain on the migration use case.
- **"All firewall migration toolkits should emit declarative formats."** This paper is specific to SonicWall → WatchGuard. Other vendor pairs may have different correct answers.

---

*Document: `docs/PAPER_why_xml_not_cli.md`*
*Companion docs: `OPERATIONS_MANUAL.md`, `docs/DEMO_ike_vpn_translation.md`, `docs/DEMO_llm_provider_subsystem.md`*
*Toolkit version: post Phase 7 hardening*
*Reference: Fireware v2026.1.2/v12.11.8 CLI Reference (in `docs/`)*
