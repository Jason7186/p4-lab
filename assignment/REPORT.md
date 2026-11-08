# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **NNNN**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

I dumped s2's `myTunnel_exact` table and changed the entry for exit 3 (handle 2) from
port 3 to port 1: `table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1`. s1 still sends exit 3 to s2, so a packet for exit 3 bounces between the two switches. I sent one packet: `send.py 10.85.0.3 "loop" --tunnel 3`.

Right after I sent that single packet, the logs counted 1,343 packets handled by s1 and 1,345 by s2. Five seconds later the counts read 552 and 552 (`evidence/loop-before.txt`).

No. `ipv4_forward` decrements `ttl`, but nothing checks it or drops the packet at 0. `ttl` is `bit<8>`, so 0 minus 1 wraps to 255, and the packet keeps looping. A real router would drop it and send ICMP Time Exceeded. The tunnel path doesn't touch the TTL at all, which is why tunnelled packets arrived with TTL 64.

## 2. Design: the hop limit

- Size: `bit<8>`. Up to 255 hops is plenty, it matches IPv4's TTL. The tunnel header grows  from 4 to 5 bytes.
- Position: after `dst_id`. `proto_id` and `dst_id` keep their old offsets, so a switch running the old program can still read them and forward by `dst_id`.
- Start value: 8. The longest real path crosses 2 switches, so 8 leaves margin but kills a loop quickly. `send.py --hops` can change it.
- Decrement and drop: in ingress, before `myTunnel_exact`. If `hop_limit` is 0 the packet is dropped. Otherwise it is decremented and the table is applied. The check comes first because 0 − 1 would wrap to 255. The table runs only in the `else` branch, so it can't undo the drop. Starting at 8, a looping packet is processed 9 times.
- Agreement: the parser and deparser need no new code, since extract and emit cover the whole header. `send.py` and `receive.py` both add `ByteField("hop_limit")`, in the same position, because Step 6 showed that hosts and switches must agree on the header's layout.

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

## AI use

Level 3

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
