# OpenStack Per-VM 1 Gbps QoS Using Linux TC + IFB

## 1. Purpose

This SOP explains how to apply a **1 Gbps TX/RX bandwidth limit to one specific OpenStack VM** from its compute host.

This method is useful when only a small number of VMs, for example 5–10 VMs, need bandwidth restrictions and you do not want to change the global OpenStack/Neutron configuration.

### Target result

```text
VM TX → maximum ~1 Gbps
VM RX → maximum ~1 Gbps
```

The policy is applied only to the selected VM's virtual network interface.

---

# 2. Important Architecture

In this example:

```text
VM: QoS1-1
VM IP: 10.15.20.86
Destination VM: 10.15.20.180
Compute host: compute04
```

The VM's Neutron port is:

```text
8a5d25d1-c9f2-4627-84f1-da2500b164cc
```

Its host-side tap interface is:

```text
tap8a5d25d1-c9
```

The relevant network path on the compute host is approximately:

```text
QoS1-1 VM
    |
    | virtual NIC
    |
    v
tap8a5d25d1-c9
    |
    v
qbr8a5d25d1-c9
    |
    v
qvb / qvo
    |
    v
OVS br-int
    |
    v
OpenStack network
```

The `tap` interface is effectively the host-side connection for that VM's virtual NIC.

---

# 3. Very Important: TX and RX Direction

This is the most important concept in this SOP.

A Linux interface has two traffic directions:

```text
Ingress = traffic entering the interface
Egress  = traffic leaving the interface
```

For our VM tap interface:

```text
                 VM
                  |
                  |
        +---------+---------+
        |                   |
       TX                  RX
        |                   |
        v                   ^
     tap ingress        tap egress
        |                   |
        v                   |
       IFB              TBF 1G
        |
      TBF 1G
```

Therefore:

### VM RX

Traffic:

```text
Network → tap → VM
```

This can be shaped directly with:

```bash
tc qdisc add dev <tap> root tbf ...
```

### VM TX

Traffic:

```text
VM → tap → Network
```

This enters the tap interface as **ingress**.

Linux's normal root TBF operates on egress, so we cannot simply put the TBF on the tap and expect it to limit VM TX.

Instead:

```text
VM TX
  |
  v
tap ingress
  |
  v
IFB
  |
  v
TBF 1 Gbps
  |
  v
Network
```

That is why we use IFB.

---

# 4. Step 1 — Find the VM

First identify the VM and make sure it is running on the compute host where you are working.

```bash
virsh list --all
```

Example:

```text
Id    Name                State
------------------------------------
XX    instance-00000377   running
```

### Why?

We must know which VM is actually running on this compute node.

Do not apply the policy to a VM that is running on another compute host.

---

# 5. Step 2 — Find the VM's Network Interface

Run:

```bash
virsh domiflist instance-00000377
```

Example:

```text
Interface          Type      Source
--------------------------------------------
tap8a5d25d1-c9      bridge    qbr8a5d25d1-c9
```

This tells us:

```text
VM
 ↓
tap8a5d25d1-c9
 ↓
qbr8a5d25d1-c9
```

### Why?

The VM's internal interface may be called something like:

```text
eth0
```

But the compute host needs the corresponding host-side interface.

For this VM:

```text
VM:
eth0

Compute host:
tap8a5d25d1-c9
```

We apply `tc` to the **host-side tap interface**.

---

# 6. Step 3 — Check Existing Traffic Control

Before changing anything:

```bash
tc qdisc show dev tap8a5d25d1-c9
```

### Why?

This checks whether the interface already has traffic-control rules.

For a clean interface you may see:

```text
qdisc noqueue
```

If you see existing `tbf`, `htb`, `ingress`, etc., stop and investigate before adding another configuration.

### Safety rule

Never blindly run:

```bash
tc qdisc add ...
```

on an interface that already has an unknown QoS configuration.

---

# 7. Step 4 — Create an IFB

Load the Linux IFB module:

```bash
modprobe ifb numifbs=1
```

### What does this do?

It loads the Linux **Intermediate Functional Block** driver.

IFB provides a virtual interface that can receive redirected traffic.

Think of it as a virtual traffic-processing interface:

```text
tap ingress
      |
      v
    IFB
      |
      v
traffic shaping
```

It does not create a physical network interface.

---

# 8. Step 5 — Check the IFB

Run:

```bash
ip link show type ifb
```

Example:

```text
ifb0: ... state DOWN
```

### Why?

We need to know which IFB is available.

In our test:

```text
ifb0
```

was available.

If `ifb0` is already being used by another VM, do not reuse it.

Use another available IFB, such as:

```text
ifb1
ifb2
```

---

# 9. Step 6 — Bring the IFB Up

Run:

```bash
ip link set dev ifb0 up
```

Verify:

```bash
ip link show dev ifb0
```

Expected:

```text
ifb0: ... state UNKNOWN
```

### Why?

The IFB must be operational before traffic can be redirected to it.

---

# 10. Step 7 — Apply RX 1 Gbps Limit

Run:

```bash
tc qdisc add dev tap8a5d25d1-c9 root tbf \
    rate 1gbit \
    burst 1mb \
    latency 50ms
```

## What is happening?

We attach a **TBF (Token Bucket Filter)** to the tap interface's root/egress path.

The important parameter is:

```text
rate 1gbit
```

This means the traffic is shaped to approximately:

```text
1 Gbps
```

### What is `burst`?

```text
burst 1mb
```

This controls how much data can be handled in a short burst before the configured rate becomes the limiting factor.

### What is `latency`?

```text
latency 50ms
```

This specifies how long packets may wait in the TBF queue.

---

# 11. Why This Limits VM RX

The tap's egress direction is:

```text
Network
   |
   v
tap
   |
   v
VM
```

Therefore:

```text
TBF on tap egress
        ↓
VM RX limited
```

This is why our first test worked.

---

# 12. Verify RX Configuration

Run:

```bash
tc -s qdisc show dev tap8a5d25d1-c9
```

Expected:

```text
qdisc tbf ... rate 1Gbit ...
```

The `-s` means:

```text
show statistics
```

It allows us to see:

```text
Sent
dropped
overlimits
requeues
```

---

# 13. Test RX

From the VM:

```bash
iperf3 -c 10.15.20.180 -t 10 -R
```

The `-R` is important.

Normally:

```text
iperf3 -c destination
```

means:

```text
VM → destination
```

With:

```text
-R
```

it reverses the traffic:

```text
destination → VM
```

Therefore this tests VM RX.

Expected result:

```text
~900–1000 Mbits/sec
```

Our actual 30-second test produced approximately:

```text
955 Mbits/sec
```

which confirmed the RX limit was working.

---

# 14. Step 8 — Add an Ingress Qdisc for TX

Now we need to control VM TX.

Run:

```bash
tc qdisc add dev tap8a5d25d1-c9 handle ffff: ingress
```

### What does this do?

It creates an **ingress qdisc** on the tap interface.

This allows us to attach a filter to traffic entering the tap.

Remember:

```text
VM TX
  |
  v
tap ingress
```

So this is where we capture VM TX traffic.

Verify:

```bash
tc qdisc show dev tap8a5d25d1-c9
```

Expected:

```text
qdisc tbf ... rate 1Gbit ...
qdisc ingress ffff:
```

The first line is RX shaping.

The second line is the TX capture point.

---

# 15. Step 9 — Redirect TX Traffic to IFB

Run:

```bash
tc filter add dev tap8a5d25d1-c9 parent ffff: protocol all u32 \
    match u32 0 0 \
    action mirred egress redirect dev ifb0
```

This looks complicated, but the purpose is simple.

### `tc filter`

Creates a traffic filter.

### `parent ffff:`

Attach the filter to the ingress qdisc created previously.

### `protocol all`

Match all traffic.

### `u32 match u32 0 0`

Match everything.

There is no IP/MAC restriction here because this tap belongs specifically to our selected VM.

### `action mirred`

Use the Linux traffic-control mirroring/redirect mechanism.

### `redirect dev ifb0`

Send the traffic to:

```text
ifb0
```

Therefore:

```text
VM TX
  |
  v
tap ingress
  |
  v
redirect
  |
  v
ifb0
```

---

# 16. Verify the Redirect

Run:

```bash
tc -s filter show dev tap8a5d25d1-c9 parent ffff:
```

You should see:

```text
mirred (Egress Redirect to device ifb0)
```

You will also see statistics such as:

```text
Sent ...
pkt ...
```

### Why are the counters important?

They prove that traffic is actually passing through the redirect.

If the counters remain at zero during an iperf test, something is wrong.

---

# 17. Step 10 — Apply TX 1 Gbps TBF

Now apply the TBF to IFB:

```bash
tc qdisc add dev ifb0 root tbf \
    rate 1gbit \
    burst 1mb \
    latency 50ms
```

The traffic path is now:

```text
VM TX
  |
  v
tap ingress
  |
  v
ifb0
  |
  v
TBF 1 Gbps
  |
  v
Network
```

This is what finally limits VM TX.

---

# 18. Verify TX TBF

Run:

```bash
tc -s qdisc show dev ifb0
```

Expected:

```text
qdisc tbf ... rate 1Gbit ...
```

During traffic you should also see increasing:

```text
Sent
overlimits
```

### What does `overlimits` mean?

It indicates that packets encountered the configured traffic-control rate/queue limitation.

It is useful evidence that the shaper is actually doing work.

---

# 19. Step 11 — Test TX

From the VM:

```bash
iperf3 -c 10.15.20.180 -t 10
```

This is the normal direction:

```text
VM → destination
```

Therefore it tests VM TX.

Expected:

```text
~900–1000 Mbits/sec
```

Our actual test produced:

```text
956 Mbits/sec sender
955 Mbits/sec receiver
```

So the TX policy was successfully enforced.

---

# 20. Final Configuration

After all steps, the VM has:

```text
                         QoS1-1
                            |
              +-------------+-------------+
              |                           |
             TX                          RX
              |                           |
              v                           ^
       tap ingress                    tap egress
              |                           |
              v                           |
           redirect                       |
              |                           |
              v                           |
            ifb0                          |
              |                           |
           TBF 1G                      TBF 1G
              |                           |
              +-------------+-------------+
                            |
                         Network
```

Therefore:

```text
TX ≈ 1 Gbps
RX ≈ 1 Gbps
```

---

# 21. How We Proved the Policy

For management/testing, use two tests.

### TX

```bash
iperf3 -c 10.15.20.180 -t 10
```

### RX

```bash
iperf3 -c 10.15.20.180 -t 10 -R
```

Expected:

```text
TX ≈ 950 Mbps
RX ≈ 950 Mbps
```

The exact result does not need to be exactly `1000 Mbps`.

For a configured 1 Gbps Linux TBF, seeing roughly:

```text
950 Mbps
```

is expected because of protocol and virtualization overhead.

---

# 22. How to Prove the Policy Is Actually Active

Do not rely only on iperf.

Use both:

### Configuration evidence

```bash
tc -s qdisc show dev <TAP>
```

and:

```bash
tc -s qdisc show dev <IFB>
```

### Traffic test evidence

```bash
iperf3 -c <DESTINATION> -t 10
```

and:

```bash
iperf3 -c <DESTINATION> -t 10 -R
```

### TX redirect evidence

```bash
tc -s filter show dev <TAP> parent ffff:
```

This gives you three types of evidence:

```text
1. Configuration
2. Traffic statistics
3. Actual bandwidth test
```

---

# 23. Why Other VMs Are Not Limited

Suppose compute04 has:

```text
VM-A
 └── tap-A

VM-B
 └── tap-B

VM-C
 └── tap-C
```

If you configure:

```text
tap-A → 1 Gbps
```

you are not configuring:

```text
br-int
physical NIC
bond0
Neutron network
```

Therefore:

```text
VM-A → 1 Gbps
VM-B → unchanged
VM-C → unchanged
```

This is why this method is suitable when only selected VMs need a bandwidth limit.

---

# 24. Migration Consideration

This is a **compute-host-level configuration**.

The policy is attached to:

```text
tap8a5d25d1-c9
```

not directly to the OpenStack VM object.

If the VM migrates:

```text
compute04
   |
   | migration
   v
compute05
```

the VM will get a new tap interface on compute05.

For example:

```text
compute04:
tap8a5d25d1-c9

        ↓ migration

compute05:
tapxxxxxxxx-xx
```

The old `tc` configuration remains on compute04.

Therefore you must:

```bash
virsh domiflist <VM>
```

on the new compute host and apply the policy to the new tap interface.

---

# 25. Removing the Policy

If you need to remove the policy:

## Remove RX TBF

```bash
tc qdisc del dev <TAP> root
```

## Remove TX ingress and redirect

```bash
tc qdisc del dev <TAP> ingress
```

This removes the ingress qdisc and its associated filter.

## Remove IFB TBF

```bash
tc qdisc del dev <IFB> root
```

## Bring IFB down

```bash
ip link set dev <IFB> down
```

Example:

```bash
tc qdisc del dev tap8a5d25d1-c9 root
tc qdisc del dev tap8a5d25d1-c9 ingress
tc qdisc del dev ifb0 root
ip link set dev ifb0 down
```

---

# 26. Important Cleanup Warning

Do **not** unload the IFB module blindly:

```bash
modprobe -r ifb
```

If another VM is using:

```text
ifb1
ifb2
...
```

unloading the module could affect those configurations.

First verify:

```bash
ip link show type ifb
```

---

# 27. Recommended Record for Multiple VMs

For 5–10 VMs, maintain a simple table:

| VM     | Compute   | Tap Interface  | IFB  | TX | RX |
| ------ | --------- | -------------- | ---- | -- | -- |
| QoS1-1 | compute04 | tap8a5d25d1-c9 | ifb0 | 1G | 1G |
| VM-2   | computeXX | tapXXXXXXXX    | ifb1 | 1G | 1G |
| VM-3   | computeXX | tapXXXXXXXX    | ifb2 | 1G | 1G |

This makes migration and troubleshooting much easier.

---

# 28. Quick Command Reference

For a new VM, the essential workflow is:

```bash
# 1. Find tap
virsh domiflist <VM>

# 2. Create/prepare IFB
modprobe ifb numifbs=1
ip link set dev ifb0 up

# 3. RX limit
tc qdisc add dev <TAP> root tbf rate 1gbit burst 1mb latency 50ms

# 4. TX ingress
tc qdisc add dev <TAP> handle ffff: ingress

# 5. Redirect TX to IFB
tc filter add dev <TAP> parent ffff: protocol all u32 \
    match u32 0 0 \
    action mirred egress redirect dev ifb0

# 6. TX limit
tc qdisc add dev ifb0 root tbf rate 1gbit burst 1mb latency 50ms

# 7. Verify
tc -s qdisc show dev <TAP>
tc -s qdisc show dev ifb0
tc -s filter show dev <TAP> parent ffff:

# 8. TX test
iperf3 -c <DESTINATION_IP> -t 10

# 9. RX test
iperf3 -c <DESTINATION_IP> -t 10 -R
```

---

# 29. Final Validation Criteria

Consider the configuration successful when all three conditions are true:

### RX

```text
iperf3 -R
≈ 900–1000 Mbps
```

### TX

```text
iperf3
≈ 900–1000 Mbps
```

### TC statistics

```text
TBF rate 1Gbit
```

and during testing:

```text
overlimits
```

and the ingress filter counters increase.

---

# 30. Key Concept to Remember

The entire setup can be remembered with one simple rule:

```text
VM RX:
tap egress → TBF

VM TX:
tap ingress → IFB → TBF
```

Or even simpler:

```text
RX = TBF on TAP

TX = TAP → IFB → TBF
```

This is the core of the per-VM bandwidth-limiting method used in this SOP.
