# OpenStack VM Traffic Flow & Per-VM 1 Gbps QoS

## 1. Purpose

This SOP explains:

1. How traffic travels from an OpenStack VM through the compute node.
2. What `tap`, `qbr`, `qvb`, `qvo`, and `br-int` are.
3. Why these interfaces are connected.
4. What `TC`, `TBF`, and `IFB` are.
5. How we applied a **1 Gbps TX/RX bandwidth limit to one specific VM**.
6. How we tested and verified the policy.

This procedure is designed for selected VMs only. It does not modify the global Neutron configuration.

---

# 2. Complete VM Traffic Flow

For a VM running on an OpenStack compute node using Neutron + Open vSwitch, the simplified traffic path is:

```text
                        OPENSTACK COMPUTE NODE
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   VM                                                          │
│   QoS1-1                                                       │
│   10.15.20.86                                                 │
│      │                                                        │
│      │ eth0                                                   │
│      ▼                                                        │
│   tap8a5d25d1-c9                                              │
│      │                                                        │
│      ▼                                                        │
│   qbr8a5d25d1-c9                                              │
│      │                                                        │
│      ▼                                                        │
│   qvb8a5d25d1-c9                                              │
│      │                                                        │
│      │ virtual patch connection                               │
│      ▼                                                        │
│   qvo8a5d25d1-c9                                              │
│      │                                                        │
│      ▼                                                        │
│   br-int                                                       │
│      │                                                        │
│      ▼                                                        │
│   OpenStack virtual network                                   │
│      │                                                        │
└──────┼────────────────────────────────────────────────────────┘
       │
       ▼
   VXLAN / physical network
       │
       ▼
   Destination compute node
       │
       ▼
   Destination VM
```

The exact downstream path can vary depending on the OpenStack networking topology, but the important part for this SOP is the VM-side path through the compute node.

---

# 3. What Is the VM Network Interface?

Inside the VM, you normally see:

```bash
ip addr
```

Example:

```text
eth0
```

This is the VM's virtual network card.

For QoS1-1:

```text
VM IP:
10.15.20.86
```

The VM believes it has a normal Ethernet interface.

But the physical compute server needs something to connect that virtual NIC to the OpenStack network.

That is where the TAP interface comes in.

---

# 4. What Is TAP?

Example:

```text
tap8a5d25d1-c9
```

A TAP interface is a **virtual network interface on the Linux compute host**.

You can think of it as:

> The virtual network cable between the VM and the compute host.

Conceptually:

```text
VM                         Compute Host

eth0  ===================  tap8a5d25d1-c9
       virtual connection
```

The VM does not normally see:

```text
tap8a5d25d1-c9
```

The compute host sees it.

For QoS1-1, this is the interface we used for bandwidth control.

---

# 5. What Is QBR?

Example:

```text
qbr8a5d25d1-c9
```

`qbr` is a **Linux bridge**.

A Linux bridge behaves like a small virtual Ethernet switch.

Its job is to connect the VM-side TAP interface to the OVS networking side.

Simplified:

```text
tap
 |
 v
qbr
 |
 v
OVS side
```

Think:

```text
TAP = VM cable
QBR = small virtual switch
```

---

# 6. What Are QVB and QVO?

You will normally see:

```text
qvb8a5d25d1-c9
qvo8a5d25d1-c9
```

These form a virtual connection between the Linux bridge and Open vSwitch.

Think of them as two ends of a virtual Ethernet cable:

```text
qvb ================= qvo
       virtual link
```

### QVB

`qvb` is on the Linux bridge side:

```text
qbr
 |
qvb
```

### QVO

`qvo` is on the Open vSwitch side:

```text
qvo
 |
br-int
```

So:

```text
tap
 |
qbr
 |
qvb
 ||
 qvo
 |
br-int
```

---

# 7. What Is BR-INT?

`br-int` means:

> Open vSwitch Integration Bridge

It is one of the important OVS bridges used by Neutron.

It connects VM network ports to the OpenStack virtual networking system.

Simplified:

```text
VM
 |
tap
 |
qbr
 |
qvb/qvo
 |
br-int
 |
OpenStack network
```

Think of:

```text
br-int = main virtual switch for Neutron integration
```

---

# 8. What Is Neutron?

Neutron is the OpenStack networking service.

It manages things such as:

* Networks
* Subnets
* Ports
* IP addresses
* Security groups
* Virtual routers
* Network connectivity
* OVS networking integration

For QoS1-1, Neutron has a network port:

```text
Port:
8a5d25d1-c9f2-4627-84f1-da2500b164cc
```

MAC:

```text
fa:16:3e:04:8d:84
```

The port is associated with the VM.

The compute node then creates the corresponding networking interfaces such as:

```text
tap...
qbr...
qvb...
qvo...
```

---

# 9. Important: Does VM Traffic Go Through the Neutron Server?

Not normally.

This is an important concept.

The Neutron server is mainly the **control plane**.

For example:

```text
Horizon / CLI
      |
      ▼
Neutron API
      |
      ▼
Neutron server
      |
      ▼
Networking configuration
```

But the actual VM data traffic does not normally travel:

```text
VM → Neutron server → VM
```

Instead, the actual data path is approximately:

```text
VM
 |
tap
 |
qbr
 |
qvb/qvo
 |
br-int
 |
OpenStack network
 |
Destination VM
```

So our `tc` policy works at the compute host because that is where the VM's traffic physically enters/leaves the host's virtual networking stack.

---

# 10. What Is TC?

`tc` means:

> Linux Traffic Control

It is a Linux command used to control network traffic.

For example:

```bash
tc qdisc show dev tap8a5d25d1-c9
```

shows traffic-control configuration.

We used `tc` because we wanted to control bandwidth for **one specific VM**.

---

# 11. What Is TBF?

TBF means:

> Token Bucket Filter

It is a Linux traffic-control mechanism used for rate limiting.

We configured:

```text
rate 1gbit
```

So TBF limits the traffic to approximately:

```text
1 Gbps
```

Think of TBF as a:

> Network speed limiter.

---

# 12. Why Did We Use TBF on the TAP?

For VM RX:

```text
Network
   |
   ▼
tap
   |
   ▼
VM
```

The traffic is leaving the TAP toward the VM.

Therefore, we can put:

```text
TBF 1 Gbps
```

on the TAP egress path.

We configured:

```bash
tc qdisc add dev tap8a5d25d1-c9 root tbf \
    rate 1gbit \
    burst 1mb \
    latency 50ms
```

This controls the VM's **RX** traffic.

---

# 13. Why Didn't the Same TBF Control TX?

This is the most important part.

VM TX is:

```text
VM
 |
 ▼
tap
 |
 ▼
Network
```

From Linux's perspective, that traffic is **entering the TAP interface**.

Therefore:

```text
VM TX = TAP ingress
```

But a normal root TBF is used on the interface's egress path.

So our first TBF controlled:

```text
TAP → VM
```

which is RX.

It did not control:

```text
VM → TAP
```

which is TX.

---

# 14. What Is IFB?

IFB means:

> Intermediate Functional Block

It is a virtual Linux network interface used with `tc`.

We created:

```text
ifb0
```

The purpose was to take traffic entering the TAP and redirect it to another virtual interface where we could apply the TBF.

The TX path becomes:

```text
VM
 |
 ▼
tap ingress
 |
 ▼
ifb0
 |
 ▼
TBF 1 Gbps
 |
 ▼
Network
```

So IFB is basically a:

> Virtual traffic-processing detour.

---

# 15. Why Did We Add the Ingress Qdisc?

We ran:

```bash
tc qdisc add dev tap8a5d25d1-c9 handle ffff: ingress
```

This creates an ingress traffic-control point on the TAP.

It allows us to inspect and redirect traffic entering the TAP.

Remember:

```text
VM TX
 |
 ▼
TAP ingress
```

So this is where we capture VM TX traffic.

---

# 16. Why Did We Use the `mirred` Rule?

We ran:

```bash
tc filter add dev tap8a5d25d1-c9 parent ffff: protocol all u32 \
    match u32 0 0 \
    action mirred egress redirect dev ifb0
```

The important part is:

```text
redirect dev ifb0
```

It means:

> Redirect traffic entering this TAP interface to IFB0.

The resulting path is:

```text
VM TX
  |
  ▼
tap ingress
  |
  ▼
redirect
  |
  ▼
ifb0
```

Then we put TBF on IFB:

```bash
tc qdisc add dev ifb0 root tbf \
    rate 1gbit \
    burst 1mb \
    latency 50ms
```

So now:

```text
VM TX
  |
  ▼
tap ingress
  |
  ▼
ifb0
  |
  ▼
TBF 1 Gbps
  |
  ▼
Network
```

---

# 17. Final QoS Configuration

For QoS1-1, our final configuration was:

```text
                       QoS1-1 VM
                      10.15.20.86
                           |
                         eth0
                           |
                           ▼
                  tap8a5d25d1-c9
                           |
             +-------------+-------------+
             |                           |
           RX path                    TX path
             |                           |
             ▼                           ▼
         TBF 1 Gbps                ingress
             |                           |
             ▼                           ▼
            VM                         IFB0
                                       |
                                       ▼
                                  TBF 1 Gbps
                                       |
                                       ▼
                                    Network
```

### RX

```text
Network → TAP → TBF → VM
```

### TX

```text
VM → TAP → IFB → TBF → Network
```

---

# 18. Commands We Actually Used

## RX policy

```bash
tc qdisc add dev tap8a5d25d1-c9 root tbf \
    rate 1gbit burst 1mb latency 50ms
```

## Create IFB

```bash
modprobe ifb numifbs=1
```

## Enable IFB

```bash
ip link set dev ifb0 up
```

## Capture TAP ingress

```bash
tc qdisc add dev tap8a5d25d1-c9 handle ffff: ingress
```

## Redirect TX to IFB

```bash
tc filter add dev tap8a5d25d1-c9 parent ffff: protocol all u32 \
    match u32 0 0 \
    action mirred egress redirect dev ifb0
```

## TX policy

```bash
tc qdisc add dev ifb0 root tbf \
    rate 1gbit burst 1mb latency 50ms
```

---

# 19. How We Tested It

## TX test

From QoS1-1:

```bash
iperf3 -c 10.15.20.180 -t 10
```

This tests:

```text
QoS1-1 → QoS1-2
```

Expected:

```text
~1 Gbps
```

Our 30-second validation produced approximately:

```text
956 Mbps
```

---

## RX test

From QoS1-1:

```bash
iperf3 -c 10.15.20.180 -t 10 -R
```

`-R` reverses the traffic direction.

This tests:

```text
QoS1-2 → QoS1-1
```

Expected:

```text
~1 Gbps
```

Our validation produced approximately:

```text
955 Mbps
```

---

# 20. How to Verify the Policy

On the compute host:

### Check RX TBF

```bash
tc -s qdisc show dev tap8a5d25d1-c9
```

Expected:

```text
qdisc tbf ... rate 1Gbit
```

### Check TX TBF

```bash
tc -s qdisc show dev ifb0
```

Expected:

```text
qdisc tbf ... rate 1Gbit
```

### Check TX redirect

```bash
tc -s filter show dev tap8a5d25d1-c9 parent ffff:
```

Expected:

```text
mirred (Egress Redirect to device ifb0)
```

### Check actual bandwidth

```bash
iperf3 -c 10.15.20.180 -t 10
```

and:

```bash
iperf3 -c 10.15.20.180 -t 10 -R
```

Both should be around:

```text
900–1000 Mbps
```

---

# 21. Why This Does Not Affect Other VMs

The policy is attached to:

```text
tap8a5d25d1-c9
```

which belongs specifically to QoS1-1.

We did **not** configure:

```text
br-int
bond0
physical NIC
Neutron network
Neutron server
global OVS QoS
```

Therefore:

```text
QoS1-1 → 1 Gbps
VM-2   → unchanged
VM-3   → unchanged
VM-4   → unchanged
```

This is why this approach is suitable when only selected VMs need bandwidth limits.

---

# 22. What Happens If the VM Migrates?

The policy is configured on the compute host.

For example:

```text
compute04
   |
   └── QoS1-1
       └── tap8a5d25d1-c9
```

If the VM migrates:

```text
compute05
   |
   └── QoS1-1
       └── tapXXXXXXXX
```

The old `tc` configuration does not automatically move to compute05.

Therefore:

```bash
virsh domiflist <VM>
```

must be checked on the new compute host.

Then the same policy must be applied to the new TAP interface.

---

# 23. Simple Meaning of Every Term

| Term     | Simple meaning                | Why it exists                         |
| -------- | ----------------------------- | ------------------------------------- |
| VM       | Virtual machine               | Runs the workload                     |
| `eth0`   | VM's virtual NIC              | VM's network interface                |
| TAP      | Virtual network cable         | Connects VM to compute host           |
| QBR      | Linux virtual bridge          | Connects TAP to OVS side              |
| QVB      | Linux-side virtual interface  | One side of virtual connection        |
| QVO      | OVS-side virtual interface    | Connects toward OVS                   |
| `br-int` | OVS integration bridge        | Main Neutron virtual switch           |
| Neutron  | OpenStack networking service  | Manages networks, ports, IPs, etc.    |
| TC       | Linux Traffic Control         | Configures traffic shaping            |
| TBF      | Token Bucket Filter           | Limits bandwidth                      |
| IFB      | Intermediate Functional Block | Allows ingress traffic to be reshaped |
| `iperf3` | Network testing tool          | Measures actual bandwidth             |

---

# 24. Easy Mental Model

Remember this:

```text
TAP
= VM's virtual cable

QBR
= small virtual switch

QVB/QVO
= virtual cable between Linux bridge and OVS

BR-INT
= main OVS virtual switch

TBF
= bandwidth speed limiter

IFB
= virtual detour used to shape TX

TC
= tool used to configure the traffic policy

iperf3
= tool used to prove the bandwidth limit
```

---

# 25. Final Policy Flow

The entire solution can be remembered as:

```text
                    VM
                     |
                    eth0
                     |
                     ▼
              TAP interface
                     |
          +----------+----------+
          |                     |
         RX                    TX
          |                     |
       TBF 1G                ingress
          |                     |
          ▼                     ▼
         VM                    IFB
                                |
                              TBF 1G
                                |
                                ▼
                              QBR
                                |
                              QVB
                                |
                              QVO
                                |
                             br-int
                                |
                             Network
```

### Final result

```text
QoS1-1 TX ≈ 1 Gbps
QoS1-1 RX ≈ 1 Gbps
```

Only the selected VM is controlled.

No global Neutron bandwidth policy was created.

No global OVS bandwidth policy was created.

No other VM was intentionally affected.
