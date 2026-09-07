# HA/Failover Lab — Troubleshooting Log

Real issues hit during this build, documented honestly as they occurred — including one that surfaced in already-shipped code from a prior project, not just new work.

---

### 1. Firewall policy silently blocked automation access to the new switch

**Problem**
After adding the spare switch to `inventory.py`, running `topology.py` succeeded for every existing device but timed out — not an authentication failure — when connecting to the new switch.

**Diagnosis**
The existing FortiGate policy permitting the automation container (VLAN 60) to reach switch management traffic (VLAN 10) was scoped to a single-host address object (`Switch_MGMT`, hard-coded to the primary switch's IP only), not the subnet or an address range. The new switch's management IP was never covered by that policy, so the connection attempt was silently dropped at the firewall — which is exactly why it presented as a timeout rather than a credential error.

**Fix**
Rather than widen the existing object (FortiGate refuses to change an address object's type while it's actively referenced by a live policy), a second address object was created scoped to just the new device, and both objects were added to the same policy's destination list. This extended access without touching or risking the already-working rule for the primary switch.

**Why it matters**
This is least-privilege segmentation working exactly as designed, not a failure of it — the automation container doesn't get blanket access to an entire VLAN, it gets access to specifically enumerated management targets. The tradeoff is that every new managed device requires an explicit, deliberate addition to firewall policy. That's a real, disclosable limitation of a tightly-scoped security model, worth stating proactively rather than treating as a surprise.

---

### 2. The spare switch's IOS doesn't support key-based SSH authentication

**Problem**
The existing `svc-automation` service account uses RSA key-based SSH authentication on every other managed device. Attempting the identical `ip ssh pubkey-chain` setup on the spare switch failed with `% Invalid input detected`, even while genuinely in global configuration mode.

**Diagnosis**
`ip ssh ?` on the spare switch confirmed the command simply doesn't exist in this device's command tree at all. The spare switch runs an older IOS train (12.2(52)SE) that supports SSH v2 but never implemented key-based authentication for interactive sessions — a real capability gap between two switches from the same vendor and the same hardware family, not a configuration mistake.

**Fix**
Rather than force a fleet-wide assumption that broke on real hardware, `inventory.py`'s device model was used as originally designed: each entry can independently specify `use_keys` and, when `False`, a `password` field instead. No changes were needed to the automation scripts themselves — Netmiko's `ConnectHandler` already accepts either parameter natively, so the toolkit's config-driven design absorbed a real hardware limitation with a one-line data change, not a code change.

**Why it matters**
Not every device in an apparently uniform fleet supports the same automation pattern, even same-vendor, same-model-family hardware — purely due to IOS train differences. Building `inventory.py` as a flexible, per-device configuration source (rather than assuming one global auth method) turned out to matter in practice, not just in theory.

---

### 3. Fixed-width LLDP output silently produced wrong topology data

**Problem**
After enabling LLDP on the new inter-switch link, the automated topology diagram rendered the new connection with a nonsensical port label: `120 <-> GI0/1` instead of a real interface pairing.

**Diagnosis**
Cisco's `show lldp neighbors` summary table uses a fixed-width `Device ID` column. The spare switch's original hostname, `Cherwood-Financial-SW2` (22 characters), overflowed that column and ran directly into the adjacent `Local Intf` value with no whitespace separating them — for example, `Cherwood-Financial-SGi0/7` instead of two separate fields. Since the existing parser (built for Topology Discovery, a prior project) splits on whitespace, it silently misread which value belonged to which column, and the actual Hold-time value (`120`) was mistaken for a port name. No error was raised anywhere in the pipeline — the output was simply wrong, and plausible enough to be missed without cross-checking the raw source.

**Fix**
The spare switch was renamed to `Financial-SW2` (13 characters), comfortably within the column width, and matching its canonical `inventory.py` device name exactly. The pipeline was re-run and the corrected edge (`GI0/7 <-> GI0/1`) was confirmed both in the raw JSON output and on the live rendered diagram.

**Why it matters**
This is a bug in already-shipped, previously-verified code (Automated Topology Discovery) that only surfaced when new hardware with a longer hostname was introduced — a good example of why "it worked in every test so far" isn't the same as "it's correct in general." Fixed-width text parsing is fragile in a way that fails silently rather than loudly, which is a meaningfully worse failure mode than a crash: a crash gets noticed immediately, while silently wrong data can sit undetected in a diagram someone trusts. Design implication carried forward: hostnames in this environment are now deliberately kept short, and this constraint is documented rather than left as an undocumented trap for the next device added to the network.

---

*Part of the [HA/Failover Lab](./README.md) — Cherwood Financial division, Cherwood Corporation portfolio.*
