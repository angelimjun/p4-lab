# P4 Tunnel Hop-Limit Assignment

## 1. Objective

The objective of this assignment is to prevent packets from
looping indefinitely in a P4 programmable network by adding
a hop-limit mechanism to the custom tunnel header.

## 2. Implementation

I added an 8-bit `hop_limit` field to the `myTunnel_t` header
in `tunnel.p4`.

The forwarding action checks the hop limit before forwarding:

- If `hop_limit <= 1`, the packet is dropped.
- Otherwise, `hop_limit` is decreased by 1 and the packet
  is forwarded to the selected port.

I also updated `send.py` to support the `--hops` argument,
with a default value of 8.

The `receive.py` program was updated to recognize and
display the hop-limit field.

## 3. Testing

### C1: Forwarding Loop Without Hop Limit

I modified the forwarding rule on s2 so that packets with
`dst_id=3` were sent back to s1 instead of h3.

The packet continued looping between s1 and s2.

The switch logs contained 568 lines for s1 and 566 lines
for s2 during the loop experiment. These repeated log entries
provide evidence of continuous forwarding activity between
the switches. The values are log-line counts, not exact
packet-forwarding counts.

Evidence: `assignment/evidence/loop-before.txt`

### C4: Forwarding Loop With Hop Limit

I repeated the same forwarding-loop experiment using
the updated P4 program.

The test packet was sent with `hop_limit=8`.

The switch log-line counts remained unchanged
between measurements approximately 5 seconds apart:

- s1: 328
- s2: 326

This shows that the packet was no longer looping continuously,
consistent with the hop-limit protection working correctly.

Evidence: `assignment/evidence/loop-after.txt`

### Normal Forwarding

The updated program successfully delivered a tunnel packet
to h2 with `dst_id=2` and a remaining `hop_limit=6`.

A tunnel packet was also successfully delivered to h3
with `dst_id=3` and a remaining `hop_limit=6`.

Both tunnel delivery tests started with `hop_limit=8`,
confirming that the two switches decremented the hop limit.

Normal IPv4 forwarding was tested using three ICMP
echo requests from h1 to h2. All three replies were
received, with 0% packet loss.

Evidence:
- `assignment/evidence/delivery-h2.txt`
- `assignment/evidence/delivery-h3.txt`
- `assignment/evidence/ipv4-ping.txt`

## 4. C5: Mixed-Version Rollout

I tested a mixed-version topology:

- s1 used the new `tunnel.p4` with hop limit.
- s2 used the old `tunnel_v1.p4` without hop limit.

### Prediction

I predicted that the old switch might interpret the new
tunnel packet incorrectly because of the extra 8-bit field.

### Observation

Normal IPv4 ping succeeded with 0% packet loss.

The old s2 matched the tunnel forwarding rule for `dst_id=2`
and transmitted the packet through port 2.

However, the h2 receiver application did not report the packet.

A tcpdump capture confirmed that the packet reached h2.
The captured bytes included `0800 0002 07 45`, where:

- `0800` is the protocol ID.
- `0002` is the destination ID.
- `07` is the remaining hop limit.
- `45` is the beginning of the IPv4 header.

The old switch expected IPv4 immediately after `dst_id`.
Therefore, it interpreted the packet with a one-byte
header misalignment.

This shows that changing a packet header format can create
compatibility problems during a mixed-version rollout.

Evidence: `assignment/evidence/mixed-capture.txt`

### Mixed-Version Forwarding Loop Test

I also tested a forwarding loop with s1 running the new
`tunnel.p4` and s2 running the old `tunnel_v1.p4`.

I modified the s2 forwarding rule for `dst_id=3` to use
port 1, sending the packet back to s1.

I sent a tunnel packet with an initial `hop_limit=8`.

The s1 log showed seven hop-limit decrements followed by
`mark_to_drop` and `Dropping packet at the end of ingress`.

The switch log-line counts were:

| Switch | First measurement | Second measurement |
|---|---:|---:|
| s1 | 14268 | 14268 |
| s2 | 14182 | 14182 |

The counts remained unchanged between measurements.

This confirms that the upgraded s1 dropped the looping
packet even though s2 was running the old P4 program.

However, the earlier delivery test showed that mixed-version
operation still caused a tunnel-header compatibility problem.

Evidence:
- `assignment/evidence/mixed-loop.txt`
- `assignment/evidence/mixed-loop-s1.txt`

## 5. Safe Rollout Strategies

### Strategy 1: Coordinated Update

Update both switches to the new P4 program together.

This reduces the time when incompatible versions are running,
but packets in transit may still be affected during the update.

### Strategy 2: Versioned Tunnel Header

Add a version identifier and support both old and new tunnel
formats during migration.

This allows switches to recognize different packet formats,
but requires additional parsing and forwarding logic.

## 6. Data Plane vs. Control Plane

The hop-limit mechanism works in the data plane.
Each switch checks and decreases the hop limit as packets
pass through it.

The Week 3 `check_paths()` approach works in the control plane.
It checks network paths before forwarding rules are installed.

Control-plane checking can prevent incorrect configurations,
while the data-plane hop limit provides protection when
forwarding loops still occur.

Both approaches can be used together.

## 7. Hardware Trade-offs

An 8-bit hop limit adds one byte to each tunnel packet.

Each switch must also perform a comparison and subtraction
before forwarding.

These operations are simple, but they still use processing
resources in the programmable data plane.

The benefit is that forwarding loops can be stopped without
waiting for the control plane to detect and correct them.

## 8. AI Assistance Declaration

I used ChatGPT (OpenAI) as a learning and coding assistant
during this assignment.

AI assistance included explaining the P4 tunnel implementation,
suggesting code changes for the hop-limit mechanism, guiding
the Mininet tests, interpreting switch logs and packet captures,
and helping organize and draft this report.

I ran the commands, tested the implementation, and reviewed
the observed results. The report describes the results obtained
from my experiments.
