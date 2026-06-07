# SonicWall Portshield → WatchGuard Bridge Migration Playbook

This document is the full operator guide for migrating SonicWall
portshield configurations to WatchGuard Bridge interfaces. The
xml_emitter generates a brief migration note when portshield is
detected; the full guide lives here.

## What is portshield?

SonicWall's **portshield** is a port-aggregation feature. Multiple
physical ports (e.g. X4, X5) are bonded to a "parent" port (e.g.
X0). The bonded ports share the parent's IP and zone, acting
together as a single Layer-2 broadcast domain — essentially an
unmanaged switch built into the firewall.

Example from a SonicWall config:

```
interface X4
    ip-assignment LAN portshield X0
    ...
```

This means: X4 is in the LAN zone and bonded to X0 (the parent).
Hosts plugged into X4 behave as if plugged into X0.

## What's the WatchGuard equivalent?

WatchGuard's equivalent is a **Bridge interface** — a virtual
interface that combines two or more physical interfaces into a
single Layer-2 segment. The Bridge has its own zone assignment,
and the underlying physical interfaces become "Bridge members"
that don't carry their own IPs.

Architectural differences worth knowing:

- **Schema**: SonicWall uses an `ip-assignment` modifier on the
  child interface. WG uses a separate `<interface>` element with a
  Bridge type, plus `<bridge-member-list>` referencing the
  underlying physical interfaces.
- **Zone semantics**: Both products attach a zone to the bonded
  group. SonicWall inherits from the parent; WG attaches to the
  bridge directly.
- **L2 loop risk**: A misconfigured bridge can create a loop on
  the network. WG requires explicit operator action to create one,
  by design.

## Why doesn't this toolkit auto-generate the bridge?

Three reasons:

1. **Schema gap.** The toolkit's reference WG XML export does not
   contain a bridge interface example. We have not yet captured
   the exact element shape for `<interface>`-of-type-Bridge,
   the `<bridge-member-list>`, or the relationship to the
   underlying physical interfaces. Auto-generating without a
   verified schema risks producing XML that imports incorrectly.
2. **Topology decisions.** Multiple operator choices apply:
   - Should the parent X-port also be a bridge member, or stay
     standalone with its own IP?
   - Should the bridge inherit the zone (Trusted, Optional-1, etc.)
     of the parent's merged WG interface, or get a new zone?
   - Are there VLANs on any of the bonded ports that need to
     migrate too?
3. **Verification needs operator on-site.** Bridge config can
   create L2 loops. The operator should review the resulting
   topology and connectivity before deploying.

## Step-by-step migration procedure

### Step 1: Identify the bonded group

For each portshield entry in the SonicWall config (or the migration
note), the format is:

    ip-assignment <ZONE> portshield <PARENT-PORT>

Read this as: "this interface is in <ZONE>, bonded as a member of
the group whose parent is <PARENT-PORT>."

In the slim sample, the only portshield entry was:

    interface X4
        ip-assignment LAN portshield X0

So: X4 is bonded to X0 in the LAN zone.

### Step 2: Map ports to WG dev-names

Use the SonicWall→WG interface convention this toolkit uses:

| SonicWall | WG zone | WG if-dev-name |
|---|---|---|
| X0 (LAN) | Trusted | eth1 |
| X1 (WAN) | External | eth0 |
| X2 (DMZ) | Optional-1 | eth2 |
| X3 (VPN) | Optional-2 | eth3 |
| X4 (varies) | (per zone) | eth4 |

So in the slim example: the SonicWall bonded group `[X0, X4]` in
zone LAN maps to WG ports `[eth1, eth4]` in zone Trusted.

### Step 3: Configure the bridge in Firebox

In the Firebox web UI:

1. Navigate to **Network → Configuration → Interfaces**.
2. The current generated XML has Trusted (eth1) populated with the
   real X0 IP. **Note this IP** — you will move it to the bridge
   in Step 4.
3. Click **New** → choose **Bridge** as the interface type.
4. Give the bridge a name (e.g. `Trusted-Bridge` or just `Trusted`
   if you delete the existing Trusted interface).
5. Add the relevant `ethN` interfaces as bridge members. For the
   slim example: add **eth1** (was X0) and **eth4** (was X4).
6. Assign the same zone the bonded group had on SonicWall. For
   slim: **Trusted**.

### Step 4: Move the IP from physical to bridge

If the parent port had its IP merged into a canonical zone
interface (Trusted, External, etc.) by the toolkit, that IP is now
on a **physical** interface that will become a bridge member. WG
requires the IP to be on the bridge itself, not on a member.

1. Note the IP/netmask/gateway from the merged Trusted/External/
   Optional-N interface.
2. **Remove** the IP/netmask from that physical interface (which is
   about to become a bridge member).
3. **Set** the IP/netmask/gateway on the new Bridge interface.
4. The bridge is now the L3 gateway for hosts on either bonded
   port.

### Step 5: Verify connectivity

1. Connect a test host to one of the bonded ports (e.g., a port
   that maps to X4).
2. Confirm it gets DHCP / static IP in the bridge subnet.
3. Confirm it can reach the bridge IP (acting as gateway).
4. Confirm it can reach external networks (passes through firewall
   policies as expected).
5. Repeat for each bonded port.

### Step 6: Adjust firewall policies if needed

Most policies that used the SonicWall LAN zone should automatically
work because the bridge inherits the zone. Sanity-check:

- Policies referencing **Trusted** (or External, Optional-N for
  other bonded groups) should now apply to traffic from any of the
  bonded ports.
- Address objects referencing X0 directly may need updating (use
  the bridge name or the zone name instead).

## Common variations

**Multiple portshield groups**: A SonicWall may have several
portshield groups (e.g., X4→X0 in LAN AND X5→X1 in WAN). Each
group becomes a separate WG bridge.

**Portshield with VLANs**: If any bonded port also has VLAN
sub-interfaces, those VLANs must migrate too. Each VLAN becomes a
separate WG VLAN interface attached to the bridge.

**Mixed-zone confusion**: SonicWall lets you accidentally
portshield ports across zones (rare but possible). WG bridges are
strictly single-zone. Triple-check the source for this case.

## Verification with the validator

After manually adding the bridge to the configuration.xml and
re-running the migration toolkit:

```
python3 wg_validator/wg_validator.py --candidate configuration.xml
```

A correctly-configured bridge should validate cleanly. A
misconfigured one will likely produce kind-shape or required-fields
warnings — review the validator output before deploying.

## Future automation

Once the toolkit captures a reference WG XML export containing a
bridge interface, the schema can be extended and a second pass of
this document will likely make most of the manual steps automatic.
Until then, the migration note + this playbook are the operator's
guide.
