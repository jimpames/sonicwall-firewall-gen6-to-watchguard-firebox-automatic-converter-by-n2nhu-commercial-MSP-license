# Side-by-Side: One IKEv2 VPN, Two Vocabularies

## What you're looking at

The same site-to-site IKEv2 VPN tunnel. Captured in the source SonicWall CLI export. Emitted into the target WatchGuard XML config. **Same tunnel, completely different vocabularies.** The toolkit bridges them.

---

## ◀ SonicWall NSA 3600 CLI (the source) | ▶ WatchGuard T30 XML (the target)

### Basic identity & state

| SonicWall | WatchGuard XML |
|---|---|
| `vpn policy site-to-site fakesite2site` | `<ike-policy>` <br> `  <name>fakesite2site</name>` |
| `enable` | `<enabled>1</enabled>` |
| `bound-to zone WAN` | `<local-if>External</local-if>` |

### Peer identity

| SonicWall | WatchGuard XML |
|---|---|
| `gateway primary fakeprigwtun` | `<peer-addr>14.14.14.14</peer-addr>` |
| `ike-id local ipv4 192.168.1.254` | `<local-id-data>192.168.1.254</local-id-data>` <br> `<local-id-type>1</local-id-type>` |
| `ike-id peer ipv4 14.14.14.14` | `<peer-auth>` <br> `  <peer-auth-mask>1</peer-auth-mask>` <br> `  <ip-addr>14.14.14.14</ip-addr>` <br> `</peer-auth>` |

### Authentication

| SonicWall | WatchGuard XML |
|---|---|
| `auth-method shared-secret` | `<auth-method>1</auth-method>` |
| `shared-secret 4,afb1abf8a107b4b193...` | `<psk>+1234abcdef...</psk>` <br> `<psk-hex>0</psk-hex>` |

### Phase 1 (IKE) cryptographic proposal

| SonicWall | Abstract | WatchGuard XML |
|---|---|---|
| `proposal ike exchange ikev2` | IKE Version 2 | `<version>2</version>` |
| `proposal ike encryption aes-128` | AES-128 | `<encryp-algm>5</encryp-algm>` <br> `<encryp-key-len>16</encryp-key-len>` |
| `proposal ike authentication sha-1` | SHA-1 | `<auth-algm>2</auth-algm>` |
| `proposal ike dh-group 2` | DH Group 2 | `<dh-group>2</dh-group>` |
| `proposal ike lifetime 28800` | 28,800 seconds = 8 hours | `<lifetime>8</lifetime>` <br> `<time-unit>1</time-unit>` |

### Phase 2 (IPsec) cryptographic proposal

| SonicWall | Abstract | WatchGuard XML |
|---|---|---|
| `proposal ipsec protocol esp` | ESP | (encoded in `<ipsec-action>`) |
| `proposal ipsec encryption aes-128` | AES-128 | `<encryp-algm>5</encryp-algm>` |
| `proposal ipsec authentication sha-1` | SHA-1 | `<auth-algm>2</auth-algm>` |
| `no proposal ipsec perfect-forward-secrecy` | PFS off | `<pfs>0</pfs>` |
| `proposal ipsec lifetime 28800` | 8 hours | `<lifetime>8</lifetime>` <br> `<time-unit>1</time-unit>` |

### Protected networks

| SonicWall | WatchGuard XML |
|---|---|
| `network local group "LAN Subnets"` | `<network-group>LAN Subnets</network-group>` |
| `network remote name someremotepeerfake` | `<network-name>someremotepeerfake</network-name>` |

---

## Why it looks intimidating but isn't

Three things make the WatchGuard side feel cryptic compared to SonicWall:

**1. Numeric encoding instead of human-readable values.**
SonicWall says `aes-128`. WatchGuard says `encryp-algm = 5`. There IS a lookup table:

```
encryp-algm:                         auth-algm:
  3 = 3DES                            1 = MD5
  5 = AES-128                         2 = SHA-1
  6 = AES-192                         4 = SHA-256
  7 = AES-256                         5 = SHA-384
                                      6 = SHA-512
```

The toolkit captures this mapping ONCE in `xml_emit_config.ini`. After that, every migration uses it automatically.

**2. Nested XML structures instead of flat indentation.**
SonicWall flattens everything under the `vpn policy` block. WatchGuard splits the same information across four cross-referenced lists:
- `<ike-action-list>` — the cryptographic transforms
- `<ike-policy-list>` — the peer gateway definitions
- `<ike-policy-group-list>` — the policy-to-gateway bindings
- `<bovpn-tunnel-list>` — the actual tunnel with subnets

**3. Unit conversion (seconds → hours).**
SonicWall: `lifetime 28800` (seconds). WatchGuard: `lifetime = 8` + `time-unit = 1` (hours). The toolkit does the arithmetic.

---

## The cheat sheet

If you remember nothing else from this slide:

```
SonicWall CLI text          →   Abstract Concept           →   WatchGuard XML
─────────────────────────       ─────────────────────────      ─────────────────────────
human-readable nouns            vendor-neutral fields          nested cross-referenced XML
flat indentation                flat record                    four separate <*-list> blocks
aes-128 (string)                AES_128 (enum)                 <encryp-algm>5</encryp-algm>
seconds                         seconds                        hours
"on by default" if absent       boolean field                  explicit 0 or 1
```

The toolkit owns the right column transformation. We never have to think about WatchGuard's numeric codes again.

---

*Pull this up next to `DEMO_ike_vpn_translation.md` for the full deep-dive. This file is for the 10-minute version.*
