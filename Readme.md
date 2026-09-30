# 🔒 Per-VM 1 Gbps Bandwidth Limit with Linux TC + IFB

> **Purpose:** Apply a 1 Gbps bandwidth limit to a **specific OpenStack VM** without changing global Neutron, OVS, or other VM configurations.

## ⚠️ Important

This is a **manual compute-node configuration**.

* Applies only to the selected VM's tap interface.
* Does not change Neutron configuration.
* Does not change global OVS configuration.
* Does not affect other VMs.
* The policy does **not automatically follow VM live migration**.
* After migration to another compute node, the policy must be applied again to the VM's new tap interface.
* Test carefully on production systems.

---

# 1. Traffic Direction

Linux `tc` works on interface traffic direction.

For this setup:

```text
VM TX
  │
  ▼
tap interface ingress
  │
  ▼
IFB
  │
  ▼
TBF 1 Gbps
  │
  ▼
Network
```

For VM RX:

```text
Network
  │
  ▼
tap interface egress
  │
  ▼
TBF 1 Gbps
  │
  ▼
VM
```

Therefore:

* **RX:** TBF directly on the VM tap interface.
* **TX:** tap ingress → IFB → TBF.

---

# 2. Before Starting

SSH to the **compute host where the VM is currently running**.

Find the VM:

```bash
virsh list --all
```

Find its interface:

```bash
virsh domiflist <VM_NAME>
```

Example:

```bash
virsh domiflist instance-00000377
```

Example output:

```text
Interface       Type      Source              Model
-------------------------------------------------------
tap8a5d25d1-c9  bridge    qbr8a5d25d1-c9      virtio
```

Record the tap interface:

```text
TAP=tap8a5d25d1-c9
```

> **Do not guess the tap interface. Always verify it with `virsh domiflist`.**

---

# 3. Safety Check

Before changing anything:

```bash
tc qdisc show dev $TAP
```

Also check whether the tap already has an ingress qdisc:

```bash
tc qdisc show dev $TAP
```

Check IFB interfaces:

```bash
ip link show type ifb
```

### Important

If the VM already has existing `tc`/QoS rules, **stop and inspect them first**.

Do not blindly add another root qdisc.

---

# 4. Create an IFB for TX

Load the IFB module:

```bash
modprobe ifb numifbs=1
```

Check:

```bash
ip link show type ifb
```

If `ifb0` is available:

```bash
ip link set dev ifb0 up
```

Verify:

```bash
ip link show dev ifb0
```

Expected:

```text
ifb0: ... state UNKNOWN ...
```

> If `ifb0` is already being used by another VM, use another IFB such as `ifb1`, `ifb2`, etc.

---

# 5. Configure RX Limit — 1 Gbps

Apply TBF to the VM tap interface:

```bash
tc qdisc add dev $TAP root tbf rate 1gbit burst 1mb latency 50ms
```

Verify:

```bash
tc -s qdisc show dev $TAP
```

Expected:

```text
qdisc tbf ... rate 1Gbit ...
```

### RX test

From the VM:

```bash
iperf3 -c <DESTINATION_IP> -t 30 -R
```

Expected:

```text
Approximately 900–1000 Mbits/sec
```

For our tested VM, the result was approximately **955 Mbps**, confirming the RX limit.

---

# 6. Configure TX Ingress

Add an ingress qdisc:

```bash
tc qdisc add dev $TAP handle ffff: ingress
```

Verify:

```bash
tc qdisc show dev $TAP
```

You should see:

```text
qdisc tbf ... rate 1Gbit ...
qdisc ingress ffff:
```

---

# 7. Redirect TX Traffic to IFB

Redirect all traffic entering the VM tap interface to the IFB:

```bash
tc filter add dev $TAP parent ffff: protocol all u32 \
    match u32 0 0 \
    action mirred egress redirect dev ifb0
```

Verify:

```bash
tc -s filter show dev $TAP parent ffff:
```

You should see:

```text
action order 1: mirred
(Egress Redirect to device ifb0)
```

The packet/byte counters should increase when the VM generates traffic.

---

# 8. Apply TX 1 Gbps Limit

Apply TBF to IFB:

```bash
tc qdisc add dev ifb0 root tbf rate 1gbit burst 1mb latency 50ms
```

Verify:

```bash
tc -s qdisc show dev ifb0
```

Expected:

```text
qdisc tbf ... rate 1Gbit ...
```

---

# 9. Test TX

From the VM:

```bash
iperf3 -c <DESTINATION_IP> -t 30
```

Expected:

```text
Approximately 900–1000 Mbits/sec
```

Also check the IFB counters on the compute host:

```bash
tc -s qdisc show dev ifb0
```

You should see increasing:

```text
Sent ...
overlimits ...
```

`overlimits` increasing indicates that the TBF is actively enforcing the configured rate.

---

# 10. Final Verification

Run:

```bash
tc -s qdisc show dev $TAP
```

You should have:

```text
qdisc tbf ... rate 1Gbit ...
qdisc ingress ffff:
```

Check the TX redirect:

```bash
tc -s filter show dev $TAP parent ffff:
```

Check IFB:

```bash
tc -s qdisc show dev ifb0
```

Expected architecture:

```text
                 SPECIFIC VM
                     │
          ┌──────────┴──────────┐
          │                     │
         TX                    RX
          │                     │
          ▼                     ▲
       TAP ingress          TAP egress
          │                     │
          ▼                     │
        IFB0                    │
          │                     │
       TBF 1G                 TBF 1G
          │                     │
          └──────────┬──────────┘
                     │
                  Network
```

---

# 11. Verify Other VMs

This configuration is attached to the selected VM's tap interface.

Check another VM:

```bash
virsh domiflist <OTHER_VM>
```

Its tap interface should not have these rules.

You can verify:

```bash
tc qdisc show dev <OTHER_VM_TAP>
```

Do **not** add the rules to a physical interface, OVS bridge, or shared interface if the intention is to limit only one VM.

---

# 12. When the VM Migrates

This manual policy is attached to the **compute host's tap interface**, not the VM object itself.

Example:

```text
Before migration:

compute04
└── QoS1-1
    └── tap8a5d25d1-c9
        ├── RX TBF
        └── TX → IFB → TBF
```

After migration:

```text
compute05
└── QoS1-1
    └── tapNEW
        └── no manual TC policy
```

Therefore, after migration:

### Step 1

Find the new tap interface:

```bash
virsh domiflist <VM_NAME>
```

### Step 2

Apply the same policy to the new tap interface.

### Step 3

Test again with:

```bash
iperf3 -c <DESTINATION_IP> -t 30
```

and:

```bash
iperf3 -c <DESTINATION_IP> -t 30 -R
```

---

# 13. Remove the Policy

If you want to remove the bandwidth limit from the VM:

### Remove RX TBF

```bash
tc qdisc del dev $TAP root
```

### Remove TX ingress/redirect

```bash
tc qdisc del dev $TAP ingress
```

### Remove IFB TBF

```bash
tc qdisc del dev ifb0 root
```

Then:

```bash
ip link set dev ifb0 down
```

Verify:

```bash
tc qdisc show dev $TAP
tc qdisc show dev ifb0
```

> Do not unload the `ifb` kernel module if other VMs are using IFB interfaces.

---

# 14. Quick Checklist

Before applying to another VM:

* [ ] Confirm VM is running on this compute host.
* [ ] Run `virsh domiflist <VM>`.
* [ ] Record the correct `tap...` interface.
* [ ] Check existing `tc` configuration.
* [ ] Make sure the selected IFB is not already being used.
* [ ] Apply RX TBF.
* [ ] Add ingress qdisc.
* [ ] Add IFB redirect.
* [ ] Apply TX TBF.
* [ ] Test TX with `iperf3`.
* [ ] Test RX with `iperf3 -R`.
* [ ] Verify `tc` counters.
* [ ] Document the VM, compute host, tap interface, and IFB.

---

# 15. Example — QoS1-1

Current tested configuration:

```text
VM:
QoS1-1

VM IP:
10.15.20.86

Destination:
10.15.20.180

Compute:
compute04

Tap:
tap8a5d25d1-c9

IFB:
ifb0

RX:
1 Gbps

TX:
1 Gbps
```

Test results:

```text
TX: ~956 Mbps
RX: ~955 Mbps
```

Both directions successfully reached the expected ~1 Gbps limit.

---

## ⚠️ Production Note

This method is suitable when only a small number of VMs need temporary/manual bandwidth limits.

For your planned **5–10 selected VMs**, keep a small record such as:

```text
VM          Compute     Tap                  IFB
QoS1-1      compute04   tap8a5d25d1-c9       ifb0
VM-2        computeXX   tapXXXXXXXX          ifb1
VM-3        computeXX   tapXXXXXXXX          ifb2
```

If a VM migrates, update the record with its new compute host, tap interface, and IFB.

Do **not** apply the TBF or IFB rules to shared interfaces such as `bond0`, physical NICs, `br-int`, `qbr`, or `qvo` when the requirement is to limit **only one VM**.
