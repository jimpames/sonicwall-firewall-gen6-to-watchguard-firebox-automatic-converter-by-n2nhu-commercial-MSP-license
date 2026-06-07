# Phase 2 TODO — Schema gaps awaiting T-30 export

These four `[emit.X]` rules in `xml_emitter/xml_emit_config.ini` are
stubbed with `status = todo`. Each currently emits an XML comment marker
in the dry-run output instead of real elements.

## How to convert a TODO rule to confirmed

For each rule below:

1. **Hand-build the feature** on a Firebox (T-30 in our lab):
   - Use the WatchGuard Web UI or Policy Manager
   - Configure ONE working example with realistic but lab-safe values
   - Verify the configuration applies and works

2. **Export `configuration.xml`** from the Firebox:
   - Web UI: System → Configuration File → Export
   - Or via WSM: File → Save As

3. **Diff against an empty-baseline export** to isolate the feature's
   XML structure:
   ```bash
   diff -u baseline.xml feature.xml | less
   ```

4. **Identify the new XPath, element names, and attribute names** the
   diff reveals. Note any value-encoding choices (e.g. SHA2-256 vs
   `sha256`, hours-vs-seconds for SA lifetimes, IANA protocol numbers).

5. **Update the rule's `child_map`** in `xml_emit_config.ini`:
   - Map each canonical JSON field to the corresponding WG XML subpath
   - Add value-map references where vocabulary differs
   - Add conditional_children for fields that only sometimes appear

6. **Add value-maps to `concept_map/concept_map.ini`** for any new
   value translations encountered.

7. **Change `status = todo` → `status = confirmed`** and re-run Phase 2.

8. **Round-trip verify**:
   ```bash
   python3 xml_emitter.py --config xml_emit_config.ini
   python3 ../harness/watchparse.py ../xml_emitter/dry_run/configuration.xml
   ```
   Compare resulting Excel against your hand-built reference Excel.

---

## Rule 1: `emit.bovpn_gateway`

**SonicWall input:** `vpn_policies.json` entries with
`vpn_type == "site-to-site"`. After Phase 1.5 enrichment, these have
`phase1{}` dicts with encryption / authentication / dh_group / exchange /
lifetime_seconds.

**WatchGuard target:** `<bovpn-gateway>` element under
`/configuration/bovpn-gateway-list`.

**Lab task:** On the T-30, build a site-to-site BOVPN gateway with
PSK + AES-256/SHA2-256/DH14 + IKEv2. Export. Diff to find the exact
sub-elements for credential-method, gateway-endpoints, phase1-settings.

**Tentative mappings already drafted in INI** (verify or correct):
- `phase1.encryption` → `phase1-settings/transform/encryption`
- `phase1.authentication` → `phase1-settings/transform/authentication`
- `phase1.dh_group` → `phase1-settings/transform/diffie-hellman-group`
- `phase1.exchange` → `phase1-settings/version`

**Still missing from INI** (need export to fill):
- Pre-shared key element name and credential-method wrapper
- Local gateway endpoint (external interface, gateway ID)
- Remote gateway endpoint (peer IP, peer ID)
- IKE keep-alive / DPD / NAT-T toggles

---

## Rule 2: `emit.bovpn_tunnel`

**SonicWall input:** Same vpn_policies entries that drove the gateway,
filtered to `site-to-site`. After enrichment, has `phase2{}` dict.

**WatchGuard target:** `<bovpn-tunnel>` under `/configuration/bovpn-tunnel-list`.
References its companion gateway by name.

**Lab task:** When you build the BOVPN gateway in Rule 1, also configure
its tunnel route (a Local IP ↔ Remote IP pair). The tunnel exports
together with the gateway.

**Still missing:** Tunnel-route encoding (local/remote address pairs),
gateway reference syntax, force-key-expire seconds vs hours unit.

---

## Rule 3: `emit.mobile_vpn_ikev2`

**SonicWall input:** vpn_policies entries with `vpn_type == "group-vpn"`.
The slim sample has 2 (WAN GroupVPN, WLAN GroupVPN) — both disabled but
both flagged with deprecated 3DES/SHA-1/DH-2 crypto.

**WatchGuard target:** Mobile VPN with IKEv2 configuration. This is a
**different feature category** than BOVPN — there is no "client" object
to migrate; the SonicWall GVC client cannot connect to a Firebox at all.

**Lab task:** On the T-30, configure Mobile VPN with IKEv2 with a
RADIUS or local-user backend. Export. Identify XPath for:
- The MVPN profile/configuration
- The address-pool definition
- The crypto proposals
- The user-database reference

**Migration advisory** is already auto-generated in `migration_notes.md`.
End-users with GVC will need to install the WG IKEv2 client OR use
their OS-native IKEv2 support (Windows, macOS, iOS, Android all support
IKEv2 natively).

---

## Rule 4: `emit.nat_policies`

**SonicWall input:** `nat_policies.json`. The slim sample has 0 entries,
but real configs always have NAT.

**WatchGuard target:** Standalone NAT rules in `/configuration/nat-list`
(name TBD). `watchparse.py` shows a `policy-nat` element on individual
firewall policies but doesn't reveal the standalone NAT rule structure.

**Lab task:** On the T-30, build:
- One Dynamic NAT (SNAT) rule
- One 1-to-1 NAT rule
- One Static NAT (DNAT) rule
Export and diff. Each variety likely has a different element shape.

---

## A note on incremental delivery

You don't have to fill all four TODOs before Phase 2 is useful.
Each rule converted from `todo` to `confirmed` adds capability.
The pipeline is fully functional today for everything except these
four feature areas — and even for those, the dry-run XML contains
explicit comment markers showing exactly where the missing config
will live.

Order I'd recommend if I had to pick:
1. **NAT first** — it's the most universally used feature.
2. **BOVPN gateway + tunnel together** — they're a pair.
3. **Mobile VPN last** — biggest behavior change for end-users
   (GVC client retirement); migration project should plan for it
   separately anyway.
