# FES 1.8.3 — Release Notes

**Firewall Ejector Seat (FES) — SonicWall Gen 6 → WatchGuard Firebox config converter**

| | |
|---|---|
| **Version** | 1.8.3 — "vlan-77-wired" |
| **Release date** | June 12, 2026 |
| **Toolkit zip** | `fes_webapp_v1.8.3-vlan-77-wired.zip` |
| **Zip SHA-256** | `278f577d4271b2e1cb7f8055e9534051aa7afb597b8306d5f58eb35566e24e68` |
| **Canonical emit SHA-256** (real-world test corpus) | `fad0088b83451156a490280b3d5e7bb637ba31853b9c5fb8dd314b14f7e475ea` |
| **Target Firebox** | T-30 running Fireware 12.5.9 (also tested as schema match for T/M/cloud Firebox) |
| **Source platform** | SonicWall NSA-3600 / Gen 6.5, SonicOS 6.5.4.x — **MUST be the e-CLI (Enhanced CLI) ASCII export, captured via local FTP**. See "Producing the right SonicWall export" section below. |
| **License contact** | licensing@firewallejectorseat.com |
| **Self-service portal** | https://firewallejectorseat.com |
| **GitHub** | github.com/jimpames/sonicwall-firewall-gen6-to-watchguard-firebox-automatic-converter-by-n2nhu-commercial-MSP-license |

---

## A short preamble — read this first

This is honest engineering software, not marketing software. The notes below tell you exactly what FES does, exactly what it doesn't do, and exactly what you have to do yourself after import. Fortinet ships SSL VPN deprecation in a release note. Cisco ships hardware EOL in a release note. We ship limitations and operator-action items in a release note too. That's how engineers communicate with engineers.

If you're reading this because you're evaluating FES for production migration: read the **"Honest limitations"** and **"Operator post-import checklist"** sections first. Those tell you whether FES fits your environment.

---

## What FES does

FES converts SonicWall Generation 6 firewall configuration exports (ASCII text in SonicOS e-CLI format, captured via local FTP — see next section) into valid WatchGuard Firebox XML configuration files. The output loads on real WatchGuard hardware and renders correctly in the WatchGuard Web UI.

It is **not** a Gen 6 → Gen 7 SonicWall migration. It is an **escape hatch from the SonicWall ecosystem entirely.** Different vendor, different OS, different attack surface. The CVE-2024-40766 migration-carryover vector ceases to exist because there is no migration — the configuration is re-expressed in WatchGuard's grammar from scratch.

---

## Producing the right SonicWall export (read before uploading)

**FES only accepts the SonicWall e-CLI (Enhanced CLI) ASCII export, captured via local FTP from the firewall itself. The standard `.exp` binary export from the Web UI is NOT compatible.**

This is non-negotiable for v1.8.3. The toolkit's parser is written against the e-CLI grammar (`address-object ipv4 NAME`, `interface X2 vlan 77`, `vpn policy site-to-site NAME`, etc.) — the human-readable text format. The binary `.exp` export is an encoded blob and will fail at the first parse step.

### How to produce an e-CLI export

The exact procedure (FTP setup, CLI commands, file capture) is documented in the GitHub repo under `docs/sonicwall-export-howto.md`. The short version:

1. Enable a local FTP server on a workstation on the same network as your SonicWall
2. SSH into the SonicWall management interface
3. From SonicOS CLI, dump the running config to the FTP server in e-CLI form
4. The resulting ASCII text file is what you upload to the FES portal

**See the repo for the exact command syntax** — the SonicOS CLI export commands are version-sensitive and the repo's documented procedure is the verified one we've used in lab.

GitHub: github.com/jimpames/sonicwall-firewall-gen6-to-watchguard-firebox-automatic-converter-by-n2nhu-commercial-MSP-license

### Why the e-CLI form (not the binary `.exp`)

- **Human-readable.** Engineers can inspect what they're converting before they convert it. No black-box translation.
- **Audit trail.** The exact input text is the provenance source for the validators. Every emit decision can be traced back to a specific input line.
- **No encrypted blobs.** The binary `.exp` contains SonicOS internal-encoded credential blobs that WatchGuard can't decode anyway. The e-CLI form makes the ciphertext fields visible so operators know exactly what they'll need to rotate post-import.

---

## What's new in 1.8.3

### VLAN sub-interface support (the headline feature)

SonicWall `interface <PARENT> vlan <ID>` declarations now translate to WatchGuard VLAN-logical interfaces with full provenance:

- Parser INI extended (`parser_config.ini`) with new `[section.interfaces_vlan]` to recognize the SW VLAN sub-interface syntax
- New filler `fill_vlan_interfaces` emits the WG `<interface>` element with item-type=2 (vlan-if), proper `<vlan-id>`, member-list pointing back at the parent physical interface's `<if-num>`
- SW `comment` field carries verbatim into the WG `<description>` for full provenance
- Lab-verified: T-30 imports the VLAN cleanly and the Web UI's VLAN/Add panel renders Optional-1 as a selectable VLAN-host interface

### Validators upgraded to handle VLAN shapes

- `immutables_validator.py`: extended exemption helper now exempts both VPN-virtual interfaces (item-type=4) AND VLAN-logical interfaces (item-type=2) from STRUCTURAL_LOCK on `interface-list`. Hardware ports are still locked; logical additions are allowed.
- `schema_shape_validator.py`: added empirical exceptions for `<item><vlan-if>`, `<member><if-num>`, `<member><if-dev-name>`, `<member><pvid-enabled>` — all four shapes are in the golden WG reference but absent from the jpa training corpus.

### Unused-interface normalization

New INI file `interface_defaults.ini` defines the convention for interfaces SW didn't configure:
- Unused interfaces (Optional-2, Optional-3, etc.): `enabled=0, if-property=3, ip=0.0.0.0` — prevents jpa template's example IPs (`192.168.10.1` etc.) from leaking through
- VLAN host slot (Optional-1 by default): `enabled=1, if-property=5, ip=0.0.0.0` — VLAN-host ready, operator can add VLANs in Web UI

### Standalone user address-objects now emitted as aliases

Customer-defined SW address-objects (e.g. `address-object ipv4 myhost` with `host 1.2.3.4`) now appear in the WG Aliases panel as standalone `<alias>` entries with a backing `.1.alm` group holding the IP. Previously these were silently absorbed into consumer-group members and the standalone names were dropped.

---

## Validator state

The toolkit ships with a **10-validator gauntlet** that runs on every conversion. All ten must return exit code 0 before any output is considered shippable.

| # | Validator | Purpose | Status (canonical corpus) |
|---|---|---|---|
| 1 | `qualities_validator` | Value-type matrix vs. jpa reference (e.g. IPv4 leaf doesn't receive hostname) | ✅ 0 |
| 2 | `ref_integrity_validator` | Every named reference resolves to a defined target | ✅ 0 |
| 3 | `immutables_validator` | Hardware-locked structures (interface count, ports) unchanged | ✅ 0 |
| 4 | `customer_fidelity_validator` | Customer SW values survive into WG output | ✅ 0 |
| 5 | `schema_shape_validator` | Every emitted parent/child pair observed in real WG configs | ✅ 0 |
| 6 | `private_data_validator` | No PII or credentials leak through translation | ✅ 0 |
| 7 | `required_children_validator` | Every required child element present | ✅ 0 |
| 8 | `jpa_diff_validator` | Diff against canonical jpa reference within bounds | ✅ 0 |
| 9 | `marker_pairing_validator` | Every documented Claude-marker shape pairs SW input ↔ WG output | ✅ 0 (24/24 paired) |
| 10 | `golden_pair_validator` | Composite fidelity score against lab-verified hand-built reference | ✅ 0 (composite **0.9122 PASS**) |

**Fidelity sub-scores (canonical corpus):**
- Value fidelity (vs. SW input): **0.8333**
- Structural fidelity (vs. WG ref): **0.9910**

---

## Lab verification

Three independent lab cycles on a real WatchGuard T-30 Firebox running Fireware 12.5.9:

| Cycle | Outcome |
|---|---|
| v1.8 → v1.8.1 | First import attempted; Firebox rejected with `"400 The remote gateway IP address ... does not exist"` — peer-addr without backing address-group. Fix shipped in v1.8.1. |
| v1.8.1 → v1.8.2 | Config loaded successfully. Captain identified 4 remaining issues (fake address-object missing from emit, bogus `192.168.10.1` on Optional-2 from jpa template leak, VLAN slot not enabled, VLAN not wired). Fixes shipped in v1.8.2 and v1.8.3. |
| v1.8.3 (current) | Config loads. Web UI renders all shapes correctly: interfaces, policies, NAT, IPSec, VLAN. Captain confirmed VLAN/Add panel shows Optional-1 as VLAN-host with correct configuration. |

---

## What FES translates (full coverage list)

| SonicWall shape | WatchGuard target | Notes |
|---|---|---|
| `interface X0/X1/...` (physical) | `<interface>` with `<physical-if>` | Hardware ports preserved structurally |
| `interface X0` IPv4 LAN/WAN | Trusted/External interface IP, netmask, gateway | Customer values survive |
| `interface X<N> vlan <ID>` | `<interface>` with `<vlan-if>` + member-list to parent | New in 1.8.3 |
| `address-object ipv4 <NAME>` (host) | `<alias>` + `<address-group>.1.alm` with `<host-ip-addr>` | New in 1.8.3 — was previously dropped |
| `address-object ipv4 <NAME>` (network) | `<alias>` + `<address-group>.1.alm` with `<ip-network-addr>+ip-mask>` | New in 1.8.3 |
| `address-group ipv4 <NAME>` | `<alias>` + `<address-group>.1.alm` with members | Resolves transitive address-object references via ctx lookup |
| `service-object` | `<service>` entry | Protocol + port range |
| `service-group` | `<service>` with grouped members | |
| `schedule` | `<schedule>` entry | |
| `access-rule ipv4 from <Z1> to <Z2>` | `<abs-policy>` + `<policy>-00` + `.1.from`/`.1.to` zone aliases | 4-shape policy family |
| `nat-policy` | `<abs-policy>` + `<policy>-00` + `.1.from`/`.1.to` zone aliases + proxy/service mapping | Includes H.323/SIP/FTP proxy translation via `proxy_mapping.ini` |
| `vpn policy site-to-site` | `<ike-policy>` + `<ike-action>` (`_bo` suffix) + `<ipsec-action>` + `<ipsec-proposal>` + tunnel selector groups (`.1.tlc`/`.1.trm`) + VPN-virtual interface | Phase 1 + Phase 2 crypto translated via `wg_enums.ini` |
| IKE local ID (FQDN) | `<local-id-data>` + `<local-id-type>=2` | |
| IKE peer ID (FQDN) | `<peer-auth-mask>=4` + `<domain-name>` | |
| `gateway primary <IP>` | `<peer-addr>` + matching `<address-group>` | IP form: verbatim. See operator checklist for name form. |
| `proposal ipsec perfect-forward-secrecy` (or `no proposal`) | `<pfs>1` or `<pfs>0` | Honors SW disable marker |

---

## Honest limitations — read this before deploying

The following are **known limitations** in 1.8.3. None are bugs. They are the boundaries of what the current code knows how to do. We're publishing them rather than burying them because operators need to know what they're getting.

### Interface limitations

- **One LAN interface translated per config.** SonicWall configs with multiple LAN-zone interfaces (X0 = LAN, X4 = LAN2, X7 = LAN3, etc.) will get the **first SW LAN interface mapped to Trusted**. Additional LAN interfaces are dropped with a stderr note in the manifest. **Why:** the LAN zone-candidate chain is `Trusted, Optional-1, Optional-2, Optional-3` but Optional-1 is now reserved as the VLAN-host slot, and we haven't built logic to spill remaining LAN interfaces into other Optional slots intelligently.
- **One WAN interface translated per config.** SonicWall has multi-WAN; WG's External is one slot. Second WAN gets dropped. Multi-WAN failover on the WG side is a Web UI configuration after import.
- **One VLAN sub-interface translated per config.** If your SW config has VLAN 77 on X2 AND VLAN 88 on X3, both will currently get parented to Optional-1 (the single declared VLAN-host slot). The output will likely fail the immutables validator if it tries to add two VLAN-logical interfaces sharing the same parent. **For configs with multiple VLANs: convert one at a time today, or DM us for the multi-VLAN sprint.**
- **No logical check that physical port count fits the target Firebox.** FES targets the T-30 (5-port). If you feed it an NSA 6650 config (24 ports), FES will silently drop physical interfaces beyond the T-30's capacity. The manifest logs the drops; the operator must reconcile.
- **IPv6 interfaces are parsed but not emitted.** The parser recognizes `interface ipv6 X0` blocks (and now `interface ipv6 X2 vlan 77`) and writes them to `interfaces_ipv6.json` / `interfaces_vlan_ipv6.json`. Currently no filler reads those records. WG side gets IPv4 only.

### Site-to-site VPN limitations

- **Peer-address with DNS name (not IP) requires post-import edit.** When SonicWall has `gateway primary somehost.example.com`, FES emits the name in `<peer-addr>` and a backing address-group with placeholder `0.0.0.0` IP. The Firebox accepts the config (config loads), but the tunnel won't come up until the operator edits the gateway IP in the WG Web UI. A stderr warning prints during conversion.
- **Tunnel local CIDR currently forces /24.** When SW provides `network local network 10.5.0.0 255.255.0.0` (a /16), FES emits `10.5.0.0/24` regardless of the actual SW netmask. Operator must verify and fix the prefix in Web UI. **Saturday-coffee fix.**
- **Tunnel local/remote group/name references not resolved to underlying IPs.** When SW has `network local group "LAN Subnets"` or `network remote name <address-object>`, FES uses placeholder values for the tunnel selector. The address-object resolution that works for standalone aliases does not yet flow into tunnel selectors. **Saturday-coffee fix.**
- **PSK ciphertext carries verbatim.** SW's `shared-secret 4,<long-hex>` is the SonicOS internal-encrypted form. FES carries it through as-is; WG cannot decrypt SonicOS's internal format. The operator must replace the PSK in WG Web UI with the cleartext shared secret.
- **Secondary gateway carries through but tunnel failover policy doesn't.** SW's `gateway secondary` line is parsed but secondary-gateway IPSec failover configuration on the WG side is a Web UI step.

### Other in-scope features that are partial

- **Authentication / RADIUS / LDAP servers parsed but not emitted to WG.** SW auth server configs are caught by the parser; no filler writes them to WG. Operator configures auth servers in Web UI.
- **DHCP server configuration not migrated.** SW DHCP server, scope, reservations not translated. WG DHCP is configured in Web UI.
- **Routing policies not migrated.** SW static routes, RIP, OSPF not translated. WG routing is Web UI.
- **High-availability / failover not migrated.** SW HA pairs not translated. WG HA is Web UI.
- **SonicPoint / wireless not migrated.** Out of scope (different feature class).

### Address-object cleanup

- **SonicWall auto-generated interface aliases appear in WG output as crud.** SW creates internal address-objects for every interface (`X0 Subnet`, `X0 IP`, `MGMT Subnet`, `MGMT IP`, `X1 Default Gateway`, `X2:V77 IPv6 Addresses`, etc.). These pass through FES and appear in the WG Aliases panel. They are harmless but noisy. **Saturday-coffee cleanup INI filter is planned for 1.8.4.**

### Cleartext credential handling

- **No credential rotation.** FES does not, and cannot, rotate passwords during conversion. SonicOS-encrypted ciphertext blobs carry through where applicable. Operators MUST rotate all credentials post-import — this is also true of SonicWall's own Gen 6 → Gen 7 migration path, and is the specific failure mode behind CVE-2024-40766.

---

## Operator post-import checklist

After FES converts your SonicWall config and you import the resulting `configuration.xml` into your Firebox:

### 1. Verify the import loaded clean
- [ ] Web UI shows no "Restore Failed" error
- [ ] Interfaces panel shows your Trusted (LAN), External (WAN), and any VLAN interfaces in the expected state
- [ ] Aliases panel shows your address-objects and address-groups
- [ ] Policies panel shows your access rules and NAT rules
- [ ] VPN panel shows your IPSec site-to-site gateway and tunnel

### 2. Fix placeholder values
- [ ] **Gateway address-groups with `0.0.0.0`** — every IKE peer that was a DNS name in SW needs its gateway IP set
- [ ] **Tunnel local-side CIDR** — verify the local network address and prefix; fix /24 if your SW used a different netmask
- [ ] **Tunnel remote-side address** — if SW used `network remote name <addr-obj>` or `network remote group <...>`, the WG `.1.trm` group will need its member set to the actual remote-side IP/CIDR
- [ ] **PSK (shared secret)** — replace the SonicOS ciphertext with your actual cleartext PSK

### 3. Rotate credentials (NON-OPTIONAL)
- [ ] Rotate all local user passwords (especially anyone with SSL VPN access)
- [ ] Rotate IPSec PSKs
- [ ] Rotate any RADIUS / LDAP service account passwords
- [ ] **This is the same step SonicWall's own migration guide requires for CVE-2024-40766 mitigation. The CVE doesn't apply to WG, but the principle does.**

### 4. Configure features that don't migrate
- [ ] DHCP server scope (if you use the Firebox as DHCP server)
- [ ] Static routes and dynamic routing (RIP/OSPF/BGP)
- [ ] Authentication servers (RADIUS / LDAP / Active Directory)
- [ ] Multi-WAN failover (if applicable)
- [ ] HA / failover pair (if applicable)
- [ ] Wireless / access points
- [ ] Subscription services (Gateway AV, IPS, application control, etc.)

### 5. Cleanup
- [ ] Delete SonicWall auto-generated aliases (`X0 Subnet`, `MGMT IP`, etc.) — they're harmless but cluttery
- [ ] Disable / delete any unused interfaces that came across as Optional-2/Optional-3

### 6. Verify connectivity
- [ ] ARP / ping from Trusted side LAN to External side IP
- [ ] If using IPSec: tunnel up, ping across the tunnel
- [ ] If using VLANs: assign a host to the VLAN and verify

---

## Methodology — for engineers reading this

FES is built on **algebraic design**: complex domain rules belong in **INI configuration catalogs**, not in code branches. The principle is `Object × Verb = Action`, where every conversion decision lives in one of **36 INI files** with provenance comments naming the SonicWall source line, the WatchGuard reference XML line, or the specific lab observation that justifies the rule.

This is the methodology described in book #33, *Applied Algebraic Design for Agentic AI*. FES is the empirical proof.

Three architectural attempts:
1. Hardcoded rules — failed (brittle, exception-laden)
2. Thought-out rules — failed (couldn't scale to shape diversity)
3. INI-driven algebra — shipped

The implication for operators: **every rule FES applies is documented in an INI file with provenance**. If you find a translation decision you disagree with, you can find the INI line that made it, see the source citation, and tune it for your environment without modifying Python code.

---

## Reproducing the validator gauntlet

```bash
# Unpack the v1.8.3 zip
unzip fes_webapp_v1.8.3-vlan-77-wired.zip -d fes_webapp/
cd fes_webapp/toolkit_v11/

# Place your SonicWall config in sonicwall_parser/input/
cp /path/to/your_sw_config.txt sonicwall_parser/input/

# Run the full pipeline (parser → enricher → emitter → 10 validators)
python3 skeleton_engine/build_pipeline.py \
    --input sonicwall_parser/input/your_sw_config.txt \
    --full-rebuild

# Output:
#   skeleton_engine/output/configuration.xml  — the WG XML to import
#   Validator output printed to stdout — all 10 must show "exit: 0"
```

If any validator returns non-zero, **do not import the output to your Firebox.** Open an issue at the GitHub repo with the validator's findings attached.

---

## Acknowledgments

- **Captain Jim Ames** — methodology, architectural pivots, lab loop, every "no claim without provenance" correction
- **The T-30 Firebox at the n2nhu lab** — the final validator, three times over
- **The Akira ransomware group** — for making the timing of this release impossible to argue with

---

## Honest closing

FES is honest engineering software. It does what it does, well, with provenance you can audit. It doesn't do what it doesn't do. The list of things it doesn't do is in this document because pretending otherwise would be dishonest, and we're not in that business.

If you're an MSP managing Gen 6 fleets and need to escape the SonicWall ecosystem before the next CVE drops — FES helps you do that, with the honest caveat list above. Six-month free public beta is running now at https://firewallejectorseat.com.

If you're a network engineer who wants to read the code before trusting the math — the repo is public, the validators are in there, the golden masters are in there. Engineer the math yourself.

If you're a security insurer watching Akira account for 40%+ of your ransomware claims and 86% of those involve SonicWall — DM the licensing email. FES changes the numbers.

—n2nhu lab, Newburgh, NY
June 12, 2026
