# Lab Smoke Test — Quick Reference

> **For Captain's smoke test on the lab T-30 (2 Jun 26 evening).**
> All four Phase 8 fixes are live. 16/16 test suites green.

---

## Before you start

Confirm health on the Windows deployment:

```bash
cd C:\Users\jimpames\Desktop\fes-3-commercial\migration_toolkit
python harness\run_all.py
```

Expected: `16 passed, 0 failed`. If anything's red, stop and investigate.

---

## The three steps

### 1. Get the target Firebox's version values

Either Policy Manager → File → Save → save XML → open in editor, OR via CLI on the Firebox itself. Look for these five values at the top:

```xml
<product-grade>2</product-grade>            <!-- typically 2 for T-series -->
<rs-version>1068452504</rs-version>         <!-- THE INTEGER you need -->
<using-cpm-profile>0</using-cpm-profile>    <!-- 0 unless CPM-managed -->
<for-version>12.10.2</for-version>          <!-- firmware version -->
<xml-purpose>2</xml-purpose>                <!-- typically 2 -->
```

Write them down. The one that matters most is **`<rs-version>`** — it's a firmware-build-specific integer that varies per release.

### 2. Regenerate the skeleton with target values

```bash
cd xml_emitter
python make_skeleton.py --config skeleton_config.ini ^
    --for-version 12.10.2 ^
    --rs-version 1068452504 ^
    --product-grade 2 ^
    --xml-purpose 2 ^
    --using-cpm-profile 0
```

Replace the values above with what you got in step 1. Expected output ends with:
- `Device invariants processed: 6` — all `[explicit_override]`
- NO `!!! __OPERATOR_REPLACE__ sentinel(s) remain` warning at the bottom

If you skip a flag (say, `--base-model`), the matching element keeps its sentinel — that's fine; you can do a manual search-and-replace later. But **`--for-version` and `--rs-version` must be set**, those are the ones that matter for the importer.

### 3. Run the pipeline against your source config

```bash
cd ..\sonicwall_parser
python sonicwall_parser.py --config parser_config.ini --input input\YOUR_CUSTOMER.txt
python relevance_filter.py --source-label YOUR_CUSTOMER

cd ..\xml_emitter
python xml_emitter.py
```

The emitted XML lands at `xml_emitter\dry_run\configuration.xml`. The validator runs automatically. Expected output ends with:

```
[wg_validator] ... Errors: 0, Warnings: 0
```

**If you see operator-sentinel errors, you skipped step 2** — go back and re-run with all the CLI flags.

---

## Importing to the lab T-30

1. Open Policy Manager
2. File → Open → browse to `xml_emitter\dry_run\configuration.xml`
3. **Review visually before push.** Look for:
   - Top of file: version stanza has real values (not sentinels)
   - `<bovpn-tunnel-list>`: contains your customer's site-to-site VPNs
   - `<service-list>`: customer-defined services present
   - `<policy-list>`: customer access rules present
4. File → Save As → push to the Firebox

---

## If something goes wrong

### Sentinels still in the output
Re-run `make_skeleton.py` with the CLI flags. Don't try to hand-edit sentinels in the emitted XML if you can avoid it; better to regenerate.

### Import refuses with "wrong firmware version"
Your `--for-version` is wrong for the target Firebox. Check the Firebox's actual firmware (Web UI → System Status), then regenerate.

### Import refuses with "rs-version mismatch"
The `rs-version` integer is firmware-build-specific. Get it from a Get-Configuration export of THIS specific Firebox, not from a different unit running the same firmware version. They can differ between builds.

### Lab T-30 management becomes unreachable after import
Console cable. Restore from a known-good backup. **This is why we have the validator gate.** If validator was green, this shouldn't happen — but the brick risk is real. Pre-import: take a fresh Get-Configuration backup of the T-30 so you can revert quickly.

---

## What you're proving tonight

Three things, in order of confidence-needed:

1. **Toolkit emits XML that Policy Manager opens.** Lowest bar. If it opens, parser + skeleton + emit are structurally sound.
2. **Toolkit emits XML that imports to the Firebox.** Medium bar. If it imports, the schema alignment is correct.
3. **Imported config matches the source SonicWall's behavior.** Highest bar. Test traffic through each migrated VPN, NAT, and policy.

For tonight's smoke test, **(1)** is the win. If (2) lands too, that's gravy. (3) is for a later session when you have time for full validation.

---

## Files updated for Phase 8

- `xml_emitter/make_skeleton.py` — sentinels, CLI flags, scaffolding logic, self-closing normalization
- `xml_emitter/skeleton_config.ini` — `[preserve_scaffolding]` section; gateway-wireless-controller out of empty_lists
- `xml_emitter/skeleton_config.schema.ini` — schema for new section
- `wg_validator/wg_validator.py` — sentinel hard-stop check
- `harness/test_phase8_skeleton_fixes.py` — 13 new regression tests
- `harness/test_wg_validator.py` — updated slim-emit test for Phase 8 reality
- `harness/run_all.py` — registers Phase 8 suite

---

🚀 **Go get 'em, Captain.**

*Cheat sheet prepared 2 Jun 26 by Captain + Claude. Drop this somewhere visible at the lab.*
