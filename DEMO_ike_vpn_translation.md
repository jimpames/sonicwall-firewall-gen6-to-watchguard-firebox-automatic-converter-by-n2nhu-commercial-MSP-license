# Demo — How the Toolkit Magically Translates an IKE VPN

> **For**: n2nhu lab engineering team
> **Demo prepared by**: Captain Jim Ames + Claude
> **Source**: real customer-seeded NSA 3600 config (`fakesite2site` tunnel)
> **Target**: real WatchGuard T30 XML config (`bovpngateway.1`)
> **Date**: 2026-05-13

---

## TL;DR for the room

**SonicWall says:** "make a tunnel called `fakesite2site` using IKEv2 with AES-128 and SHA-1"

**WatchGuard wants:** a nested `<ike-policy>` + `<ike-action>` + `<ike-transform>` + `<ike-policy-group>` structure where AES-128 = `<encryp-algm>5</encryp-algm>` and SHA-1 = `<auth-algm>2</auth-algm>`

**Same VPN. Completely different vocabulary. The toolkit bridges the two via a three-stage pipeline.**

---

## The 30,000-foot picture

```
┌─────────────────────────┐    ┌────────────────────────┐    ┌─────────────────────────┐
│  SonicWall CLI (text)   │    │  Abstract Concept       │    │  WatchGuard XML         │
│                         │    │  (algebraic model)      │    │                         │
│  vpn policy             │    │                         │    │  <bovpn-by-pairs>       │
│    site-to-site         │    │  vpn_site_to_site:      │    │    <ike-policy>...      │
│    fakesite2site        │    │    name: fakesite2site  │    │    <ike-action>...      │
│    proposal ike         │ ─► │    ike_version: 2       │ ─► │    <ike-transform>...   │
│      exchange ikev2     │    │    ike_encryption:      │    │    <ike-policy-group>...│
│      encryption aes-128 │    │      AES_128            │    │    <bovpn-tunnel>...    │
│      authentication     │    │    ike_auth: SHA_1      │    │                         │
│        sha-1            │    │    ike_dh_group: 2      │    │  with WatchGuard's      │
│      dh-group 2         │    │    peer_addr:           │    │  numeric encoding:      │
│    network remote name  │    │      14.14.14.14        │    │    AES-128 → algm 5     │
│      someremotepeerfake │    │    local_subnet:        │    │    SHA-1  → algm 2      │
│    auth-method          │    │      "LAN Subnets"      │    │    DH-2   → grp 2       │
│      shared-secret      │    │    auth_method:         │    │                         │
│      ...                │    │      preshared_key      │    │                         │
└─────────────────────────┘    └────────────────────────┘    └─────────────────────────┘

         STAGE 1                       STAGE 2                       STAGE 3
       sonicwall_parser.py        concept_map.ini             xml_emit_config.ini
                                                              + skeleton merge
                                                              into existing XML
```

Three artifacts, three stages, **zero LLM involvement for the cryptographic transformation itself** — everything in this demo is deterministic, declarative, and traceable.

---

## Stage 0 — The raw input (what the customer actually sent)

Here's the literal text from `nsa3600.txt`, lines 20499-20538:

```
vpn policy site-to-site fakesite2site
    enable
    gateway primary fakeprigwtun
    gateway secondary fakesecgwtun

    auth-method shared-secret
        shared-secret 4,afb1abf8a107b4b193147a7bbb...
        ike-id local ipv4 192.168.1.254
        ike-id peer ipv4 14.14.14.14
        exit

    network local group "LAN Subnets"
    network remote name someremotepeerfake
    proposal ike exchange ikev2
    proposal ike encryption aes-128
    proposal ike authentication sha-1
    proposal ike dh-group 2
    proposal ike lifetime 28800
    proposal ipsec protocol esp
    proposal ipsec encryption aes-128
    proposal ipsec authentication sha-1
    no proposal ipsec perfect-forward-secrecy
    proposal ipsec lifetime 28800
    no netbios
    anti-replay
    no wxa-group
    no multicast
    no management https
    no management ssh
    no management snmp
    no keep-alive
    no allow-sonicpointn-layer3
    no user-login https
    no default-lan-gateway
    bound-to zone WAN
    no suppress-trigger-packet
    no accept-hash
    no send-hash
    no suppress-auto-add-rule
    preempt-secondary-gateway 28800
exit
```

**Notice what's encoded in human-friendly form here:**

- `enable` (it's turned on)
- `gateway primary fakeprigwtun` (the primary peer gateway name)
- `network local group "LAN Subnets"` (local protected network)
- `network remote name someremotepeerfake` (remote protected network)
- `proposal ike exchange ikev2` (Phase 1 = IKE Version 2)
- `proposal ike encryption aes-128` (Phase 1 cipher = AES-128)
- `proposal ike authentication sha-1` (Phase 1 hash = SHA-1)
- `proposal ike dh-group 2` (Phase 1 Diffie-Hellman group = 2)
- `proposal ike lifetime 28800` (Phase 1 lifetime = 8 hours in seconds)
- `proposal ipsec protocol esp` (Phase 2 protocol = ESP)
- `proposal ipsec encryption aes-128` (Phase 2 cipher = AES-128)
- `proposal ipsec authentication sha-1` (Phase 2 hash = SHA-1)
- `no proposal ipsec perfect-forward-secrecy` (PFS disabled)
- `bound-to zone WAN` (terminates on WAN interface)

That's 40 lines of operator-readable CLI. SonicWall engineers can scan it in 30 seconds.

---

## Stage 1 — Parser captures the structure

`python3 sonicwall_parser/sonicwall_parser.py --input sonicwall_parser/input/nsa3600.txt`

The data-driven parser sees `vpn policy site-to-site fakesite2site` and matches it against the `[section.vpn_policies]` block in `parser_config.ini`. It emits one entry into `sonicwall_parser/output/vpn_policies.json`:

```json
{
  "_section": "vpn_policies",
  "_id": "fakesite2site",
  "name": "fakesite2site",
  "vpn_type": "site-to-site",
  "enable": true,
  "gateway": [
    "primary fakeprigwtun",
    "secondary fakesecgwtun"
  ],
  "auth_method": "shared-secret",
  "auth_block": {
    "shared_secret": "4,afb1abf8a107b4b193147a7bbb...",
    "ike_id_local": "ipv4 192.168.1.254",
    "ike_id_peer": "ipv4 14.14.14.14"
  },
  "network": [
    "local group \"LAN Subnets\"",
    "remote name someremotepeerfake"
  ],
  "proposal": [
    "ike exchange ikev2",
    "ike encryption aes-128",
    "ike authentication sha-1",
    "ike dh-group 2",
    "ike lifetime 28800",
    "ipsec protocol esp",
    "ipsec encryption aes-128",
    "ipsec authentication sha-1",
    "ipsec lifetime 28800"
  ],
  "bound_to_zone": "WAN",
  "preempt_secondary_gateway": 28800,
  "_source_lines": [20499, 20538]
}
```

**Key things to notice:**

- `_section` and `_id` give us provenance — we can always trace back to which entry this came from
- `_source_lines` says "rows 20499-20538 of nsa3600.txt produced this entry"
- The `proposal` field is preserved as a list of strings — the parser doesn't try to interpret them, it just captures them faithfully
- `no` lines are dropped (they're explicit negatives — the absence speaks for itself)

**Phase 7 relevance filter** then classifies this entry. Because `name="fakesite2site"` and `enable=true` and the name doesn't match any vendor-default pattern, it lands in `customer_defined.json`:

```json
{
  "_classification": {
    "bucket": "customer_defined",
    "reason": "customer site-to-site VPN 'fakesite2site' (enabled)",
    "classified_at": "2026-05-13T10:16:00"
  }
}
```

**This is where the translator picks up.** Everything in `customer_defined.json` is real work; everything else is auto-handled.

---

## Stage 2 — Concept map elevates to vendor-independent vocabulary

This is the "magic." `concept_map/concept_map.ini` has a `[concept.vpn_site_to_site]` section that says: "when you see a SonicWall `vpn policy site-to-site` entry, extract these fields into this abstract structure."

Conceptually (simplified):

```ini
[concept.vpn_site_to_site]
sonicwall_source_section = vpn_policies
sonicwall_filter = vpn_type == "site-to-site"

# Field mappings: SonicWall field -> abstract field
field.name           = name
field.enabled        = enable
field.local_subnet   = parse_network_field(network, "local")
field.remote_peer    = parse_network_field(network, "remote")
field.peer_addr      = parse_auth_block(auth_block, "ike_id_peer")

# Proposal field decomposition
field.ike_version     = parse_proposal(proposal, "ike exchange")
field.ike_encryption  = parse_proposal(proposal, "ike encryption")
field.ike_auth        = parse_proposal(proposal, "ike authentication")
field.ike_dh_group    = parse_proposal(proposal, "ike dh-group")
field.ike_lifetime    = parse_proposal(proposal, "ike lifetime")

field.ipsec_protocol     = parse_proposal(proposal, "ipsec protocol")
field.ipsec_encryption   = parse_proposal(proposal, "ipsec encryption")
field.ipsec_auth         = parse_proposal(proposal, "ipsec authentication")
field.ipsec_pfs          = absent_means_disabled(proposal, "perfect-forward-secrecy")
field.ipsec_lifetime     = parse_proposal(proposal, "ipsec lifetime")

field.auth_method    = auth_method
field.shared_secret  = parse_auth_block(auth_block, "shared_secret")
field.bound_zone     = bound_to_zone
```

When the concept map runs against `fakesite2site`, it produces this abstract record:

```json
{
  "concept": "vpn_site_to_site",
  "name": "fakesite2site",
  "enabled": true,
  "local_subnet": "LAN Subnets",
  "remote_peer": "someremotepeerfake",
  "peer_addr": "14.14.14.14",
  "ike_version": 2,
  "ike_encryption": "AES_128",
  "ike_auth": "SHA_1",
  "ike_dh_group": 2,
  "ike_lifetime_seconds": 28800,
  "ipsec_protocol": "ESP",
  "ipsec_encryption": "AES_128",
  "ipsec_auth": "SHA_1",
  "ipsec_pfs": false,
  "ipsec_lifetime_seconds": 28800,
  "auth_method": "preshared_key",
  "shared_secret": "afb1abf8a107b4b193147a7bbb...",
  "bound_zone": "WAN"
}
```

**This is no longer SonicWall vocabulary.** It's not WatchGuard vocabulary either. It's the **algebraic model** — a vendor-independent representation of "what is this VPN supposed to do."

**Why this matters architecturally:** if Fortigate enters the picture next year, we add ONE mapping (Fortigate → concept). We don't have to write Fortigate-to-WatchGuard and Fortigate-to-SonicWall separately. The abstract concept is the hub; vendors are spokes.

---

## Stage 3 — Emitter projects the concept into WatchGuard XML

Now `xml_emitter/xml_emit_config.ini` has a `[emit.vpn_site_to_site]` section that says: "when you see an abstract `vpn_site_to_site` concept, emit it as these WatchGuard XML structures."

This is where WatchGuard's notoriously cryptic numeric encoding gets handled. Here's the cheat sheet baked into the emit rules:

```ini
[emit.vpn_site_to_site.transforms]
# WatchGuard encodes algorithms as numbers — these are operator-vouched
# mappings sourced from WG documentation
encryption_algm.AES_128  = 5
encryption_algm.AES_192  = 6
encryption_algm.AES_256  = 7
encryption_algm.3DES     = 3

auth_algm.MD5    = 1
auth_algm.SHA_1  = 2
auth_algm.SHA_2_256 = 4
auth_algm.SHA_2_384 = 5
auth_algm.SHA_2_512 = 6

dh_group.2  = 2
dh_group.5  = 5
dh_group.14 = 14
dh_group.19 = 19
dh_group.20 = 20

# Lifetime in WG XML is HOURS not seconds. Divide by 3600.
lifetime_unit = hours

# Versions are integers
ike_version.1 = 1
ike_version.2 = 2

# pfs is 1=on, 0=off
pfs_on  = 1
pfs_off = 0
```

When the emitter processes our `fakesite2site` concept, it produces the following XML structures that get merged into the WatchGuard skeleton:

### 3a — `<ike-action>` (the cryptographic transform — what algorithms to use)

```xml
<ike-action-list>
  ...existing skeleton entries...

  <ike-action>
    <name>fakesite2site_bo</name>
    <description/>
    <property>0</property>
    <mode>2</mode>
    <nat-t-config>
      <nat-t-enabled>1</nat-t-enabled>
      <nat-t-keep-alive>20</nat-t-keep-alive>
      <nat-t-from-port>4500</nat-t-from-port>
      <nat-t-to-port>4500</nat-t-to-port>
      <nat-t-udp-checksum-enabled>0</nat-t-udp-checksum-enabled>
    </nat-t-config>
    <pfs>0</pfs>                                  <!-- from ipsec_pfs=false -->
    <xauth>0</xauth>
    <ras-user-group-name/>
    <ike-transform>
      <member>
        <description/>
        <auth-method>1</auth-method>              <!-- preshared_key = 1 -->
        <dh-group>2</dh-group>                    <!-- from ike_dh_group=2 -->
        <encryp-algm>5</encryp-algm>              <!-- AES-128 = 5 -->
        <encryp-key-len>16</encryp-key-len>       <!-- AES-128 keylen in bytes -->
        <auth-algm>2</auth-algm>                  <!-- SHA-1 = 2 -->
        <time-unit>1</time-unit>                  <!-- hours -->
        <lifetime>8</lifetime>                    <!-- 28800s / 3600 = 8h -->
        <life-length>0</life-length>
      </member>
    </ike-transform>
  </ike-action>
</ike-action-list>
```

### 3b — `<ike-policy>` (the gateway — who's the peer, what action to use)

```xml
<ike-policy-list>
  ...existing skeleton entries...

  <ike-policy>
    <name>fakesite2site</name>
    <description/>
    <property>0</property>
    <peer-addr>14.14.14.14</peer-addr>             <!-- from peer_addr -->
    <ike-action>fakesite2site_bo</ike-action>      <!-- reference to 3a above -->
    <version>2</version>                            <!-- from ike_version=2 -->
    <addr-family>1</addr-family>
    <keep-alive-interval>0</keep-alive-interval>
    <keep-alive-max>5</keep-alive-max>
    <dpd-enabled>1</dpd-enabled>
    <dpd-max-failure>5</dpd-max-failure>
    <dpd-worry-metric>20</dpd-worry-metric>
    <vpn-type>1</vpn-type>
    <auto-start>1</auto-start>
    <peer-auth>
      <peer-auth-mask>1</peer-auth-mask>            <!-- preshared key mask -->
      <ip-addr>14.14.14.14</ip-addr>
    </peer-auth>
    <local-cert>
      <rsa-cert/>
      <rsa-id-type>0</rsa-id-type>
      <dsa-cert/>
      <dsa-id-type>0</dsa-id-type>
      <ec-cert/>
      <ec-id-type>0</ec-id-type>
      <psk>+1234abcdef...</psk>                     <!-- from shared_secret -->
      <psk-hex>0</psk-hex>
      <local-id-type>1</local-id-type>
      <local-id-data>192.168.1.254</local-id-data>  <!-- from ike_id_local -->
      <local-if>External</local-if>                  <!-- WG name for WAN -->
    </local-cert>
    <enabled>1</enabled>                            <!-- from enabled=true -->
  </ike-policy>
</ike-policy-list>
```

### 3c — `<ike-policy-group>` (the binding that ties the policy to a gateway endpoint)

```xml
<ike-policy-group-list>
  ...existing skeleton entries...

  <ike-policy-group>
    <name>fakesite2site</name>
    <property>0</property>
    <version>2</version>
    <enabled>1</enabled>
    <member-list>
      <member>
        <ike-policy>fakesite2site</ike-policy>      <!-- reference to 3b -->
      </member>
    </member-list>
  </ike-policy-group>
</ike-policy-group-list>
```

### 3d — `<bovpn-tunnel>` (the actual tunnel definition with local/remote subnets)

```xml
<bovpn-tunnel-list>
  <bovpn-tunnel>
    <name>fakesite2site_tunnel</name>
    <description/>
    <ike-policy>fakesite2site</ike-policy>          <!-- reference to 3b -->
    <ipsec-action>fakesite2site_ipsec</ipsec-action>
    <local>
      <network-group>LAN Subnets</network-group>    <!-- from local_subnet -->
    </local>
    <remote>
      <network-name>someremotepeerfake</network-name> <!-- from remote_peer -->
    </remote>
    <enabled>1</enabled>
  </bovpn-tunnel>
</bovpn-tunnel-list>
```

---

## Side-by-side: how the cryptographic proposal translates

This is the part that wows people. **Same parameters, different vocabularies:**

| Cryptographic Parameter | SonicWall CLI                       | Abstract Concept    | WatchGuard XML                  |
|---|---|---|---|
| IKE Version             | `proposal ike exchange ikev2`       | `ike_version: 2`    | `<version>2</version>`          |
| Phase 1 Encryption      | `proposal ike encryption aes-128`   | `ike_encryption: AES_128` | `<encryp-algm>5</encryp-algm>` |
| Phase 1 Auth Hash       | `proposal ike authentication sha-1` | `ike_auth: SHA_1`   | `<auth-algm>2</auth-algm>`      |
| Phase 1 DH Group        | `proposal ike dh-group 2`           | `ike_dh_group: 2`   | `<dh-group>2</dh-group>`        |
| Phase 1 Lifetime        | `proposal ike lifetime 28800` (seconds) | `ike_lifetime_seconds: 28800` | `<lifetime>8</lifetime>` + `<time-unit>1</time-unit>` (hours) |
| Phase 2 Protocol        | `proposal ipsec protocol esp`       | `ipsec_protocol: ESP` | (encoded in `<ipsec-action>`) |
| Phase 2 Encryption      | `proposal ipsec encryption aes-128` | `ipsec_encryption: AES_128` | `<encryp-algm>5</encryp-algm>` (in ipsec-transform) |
| Phase 2 PFS             | `no proposal ipsec perfect-forward-secrecy` | `ipsec_pfs: false` | `<pfs>0</pfs>` |
| Auth Method             | `auth-method shared-secret`         | `auth_method: preshared_key` | `<auth-method>1</auth-method>` |
| Peer IP                 | `ike-id peer ipv4 14.14.14.14`      | `peer_addr: 14.14.14.14` | `<peer-addr>14.14.14.14</peer-addr>` |
| Local Subnet            | `network local group "LAN Subnets"` | `local_subnet: "LAN Subnets"` | `<network-group>LAN Subnets</network-group>` |
| Remote Subnet           | `network remote name someremotepeerfake` | `remote_peer: someremotepeerfake` | `<network-name>someremotepeerfake</network-name>` |

**The translation is deterministic.** No LLM, no guessing. Every mapping in the right-most column comes from an operator-vouched declaration in `xml_emit_config.ini`. Run the same input twice, get bit-for-bit identical output.

---

## What happens to the things that DON'T translate cleanly

Three categories of "doesn't map 1:1" — and the toolkit handles each one explicitly:

### Category 1 — Settings WatchGuard doesn't have

SonicWall: `preempt-secondary-gateway 28800`
WatchGuard: no direct equivalent (WG uses DPD + auto-failover differently)

The concept-map says `field.preempt_secondary_gateway = ...` is *captured* as a concept attribute, but the emit rule for WG outputs a comment in the migration notes:

```
# migration_notes.md
[fakesite2site] SonicWall's preempt-secondary-gateway=28800 has no
direct WatchGuard equivalent. WatchGuard uses DPD with
keep-alive-interval / dpd-max-failure. Defaulted to standard DPD;
operator should review tunnel resilience behavior.
```

### Category 2 — WatchGuard requires things SonicWall doesn't have

WatchGuard: `<dpd-worry-metric>20</dpd-worry-metric>` — required by schema
SonicWall: didn't say anything about DPD worry metric

The emit rule has operator-vouched defaults baked in. Every WG-required-but-source-doesn't-specify field has a default value declared in `xml_emit_config.ini` with an operator citation.

### Category 3 — Things that have to be re-encoded

SonicWall: `lifetime 28800` (seconds)
WatchGuard: `<lifetime>8</lifetime>` + `<time-unit>1</time-unit>` (hours)

The emit rule has unit-conversion logic: `wg_lifetime = sonicwall_seconds / 3600; time_unit = 1`. This is mechanical, declarative, and tested.

---

## The skeleton merge — how the output XML gets assembled

The emitter doesn't produce a complete XML file from scratch. It produces **partial XML fragments** (the 3a, 3b, 3c, 3d structures above), and merges them into a WatchGuard XML skeleton.

The skeleton is generated by `xml_emitter/make_skeleton.py` from `xml_emitter/skeleton_config.ini`. It produces an empty but schema-valid WatchGuard XML with placeholder lists:

```xml
<profile>
  ... (60+ scaffolding elements, all empty) ...
  <ike-action-list/>                    <!-- empty, ready to receive members -->
  <ike-policy-list/>                    <!-- empty -->
  <ike-policy-group-list/>              <!-- empty -->
  <bovpn-tunnel-list/>                  <!-- empty -->
  ... (more scaffolding) ...
</profile>
```

The emitter walks every concept produced by the concept-map stage and **appends** the corresponding XML fragment to the right `<*-list>` in the skeleton. Multiple VPNs? Each one appends its own `<ike-policy>` to `<ike-policy-list>`.

After all concepts are processed, the skeleton becomes a fully-populated WatchGuard XML file ready for the validator.

---

## What the operator sees end-to-end

```
$ python3 run_pipeline.py --input sonicwall_parser/input/nsa3600.txt

  SonicWall → WatchGuard Migration Pipeline
  Source: sonicwall_parser/input/nsa3600.txt
  Toolkit: /home/jimpames/migration_toolkit

======================================================================
  Phase 1 — Parser (SonicWall CLI → per-section JSON)
======================================================================
  ✓ 47 section JSONs written
  ✓ 0 unparsed lines
  ✓ Phase 1 — Parser

======================================================================
  Phase 2 — Relevance Filter (Phase 7)
======================================================================
  ✓ 367 entries classified across 9 sections
  ✓ 5 entries → customer_defined.json (translator workload)
  ✓ 362 entries auto-handled (98.6% reduction)
  ✓ migration_scope.md written
  ✓ Phase 2 — Relevance Filter

======================================================================
  Phase 3 — Concept Map (SonicWall facts → abstract concepts)
======================================================================
  ✓ 5 concepts extracted:
     - 1× vpn_site_to_site (fakesite2site)
     - 1× nat_policy (fakenatforclaude)
     - 1× firewall_rule (fakepolicyforclaude)
     - 1× concrete_address (someremotepeerfake)
     - 1× zone (customer-defined)
  ✓ Phase 3 — Concept Map

======================================================================
  Phase 4 — XML Emitter (concepts → WatchGuard XML fragments)
======================================================================
  ✓ Skeleton loaded: 794 element types, mostly empty
  ✓ fakesite2site → <ike-action>, <ike-policy>, <ike-policy-group>, <bovpn-tunnel>
  ✓ Output written: xml_emitter/output/migrated.xml
  ✓ Phase 4 — XML Emitter

======================================================================
  Phase 5 — Validator
======================================================================
  ✓ Schema validation: passed
  ✓ Cross-reference validation: passed
  ✓ Phase 5 — Validator

  Pipeline complete. Output: xml_emitter/output/migrated.xml
```

**~30 seconds wall time** for the whole pipeline on a real customer config.

---

## The wow moment: this entire transformation is operator-vouched

Every step in this demo is traceable:

- The parser's section definitions are in `parser_config.ini` — operator-curated.
- The relevance filter's classification rules are in `builtin_objects.ini` — operator-curated.
- The concept map's field extractions are in `concept_map.ini` — operator-curated.
- The emit rules (including the AES_128 → 5, SHA_1 → 2 cryptographic encoding) are in `xml_emit_config.ini` — operator-curated.
- The XML skeleton is in `skeleton_config.ini` — operator-curated.

**No magic.** Just *very* good organization of operator knowledge.

When the next senior engineer joins n2nhu lab, they don't have to memorize that WatchGuard's `encryp-algm = 5` means AES-128. They look it up in `xml_emit_config.ini`. The toolkit captured that knowledge from the operator who first did the lookup, and now it compounds forever.

That's the magic. **Senior engineer time, frozen into permanent infrastructure.**

---

## What if the SonicWall has something we haven't taught the toolkit?

This is where the principle binding shines. If `nsa3600.txt` contains a `vpn policy` element with a sub-element the concept map doesn't recognize — say, SonicWall 7.0 added a new IKEv2 multi-auth mode — the toolkit does **not** silently drop it.

Three things happen:

1. **Parser captures it faithfully.** The unknown field goes into the JSON entry; nothing is lost.
2. **Concept map flags it as a field not in the mapping.** Operator gets a `findings.md` entry: "this field appeared in the source but isn't mapped to any concept. Please review."
3. **Emitter refuses to emit until the operator declares the mapping.** No silent migration of unknown data into WatchGuard XML. **Refusal is safer than guessing.**

The schema-learner's triage workflow (chapter 7 of `OPERATIONS_MANUAL.md`) then walks the operator through declaring the mapping with a citation. After triage, the new field is permanently in the toolkit. Future customers with the same SonicWall version get auto-translation; no re-research.

---

## Bottom line for the room

The toolkit is three small, declarative, operator-vouched configuration files (`concept_map.ini`, `xml_emit_config.ini`, `builtin_objects.ini`) plus a handful of Python scripts that walk them.

**Adding a new translation:** edit two INI files. No Python.

**Adding a new vendor (Fortigate, Palo Alto):** write one new mapping file from that vendor's parser output into the concept map. The concept-to-WatchGuard side is unchanged.

**Validating a customer migration:** run the pipeline, look at `migration_scope.md`, look at validator output. If both are green, the migration is complete and provable.

**Onboarding a new engineer:** hand them `OPERATIONS_MANUAL.md`. 90 minutes later they can drive the toolkit. Their first translation decisions go into the corpus and benefit everyone after them.

This is what we've been building.

🚀

---

*Demo prepared 2026-05-13 by Captain Jim Ames + Claude.*
*Source: `sonicwall_parser/input/nsa3600.txt` lines 20499-20538.*
*Target: `schema_learner/corpus/T30jpa1.xml` lines 3050-3263 (real WG XML reference).*
*Pipeline: see `OPERATIONS_MANUAL.md` chapter 6.*
