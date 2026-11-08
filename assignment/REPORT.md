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

- `tunnel.p4`: added a `bit<8> hop_limit` field at the end of the tunnel header. If a tunnelled packet arrives with `hop_limit` 0, it is dropped. Otherwise `hop_limit` is
  reduced by 1 and the packet is forwarded.
- `send.py`: added the same field, and a `--hops` option to set the starting value (default 8).
- `receive.py`: added the same field, and prints the `hop_limit` that arrives.
- The parser and deparser didn't need new code. They now read and write 5 bytes of tunnel header instead of 4.

## 4. Evidence: it works

Delivery (`evidence/delivery.txt`): tunnelled packets sent to exit 2 and exit 3 arrived at h2 and h3 with `hop_limit=6`. They started at 8 and crossed 2 switches. `ping h3` still works, so IPv4 is unaffected.

The loop now end (`evidence/loop-after.txt`): I made the same loop as in C1. Before sending, both counts were 0. After one packet, s1 had handled 5 packets and s2 had handled 4. Five seconds later the counts were still 5 and 4, so the loop had stopped.

That is 9 passes, which matches my design. s1 saw the packet with `hop_limit` 8, 6,
4, 2 and 0, and s2 saw it with 7, 5, 3 and 1. s1 dropped it when it arrived with 0.
In C1 the same loop was still running at around 1,000 packets per second.

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

a) Prediction: I expect s2 to still send the packet to h2, because my new byte comes after `dst_id`, so the old program reads `dst_id` correctly. But s2 only knows a 4-byte tunnel header, so it will treat my `hop_limit` byte as the start of the IPv4 header, and read IPv4 one byte off. When it then recalculates the IPv4 checksum, it may overwrite the real protocol byte, so h2's `receive.py` might not show the packet even though it arrives. The loop should still end, but take about twice as long (around 17 passes instead of 9), because only s1 decreases the hop limit.

What happened: As predicted. s2 read `dst_id` correctly and sent the packet to h2, but `receive.py` on h2 showed nothing. The tcpdump capture shows why, the packet arrived with `hop_limit` 07 (s1 decreased it, s2 didn't), but the IPv4 protocol byte was `0d` instead of `11` (UDP). s2 had read IPv4 one byte off, so its checksum update overwrote the real protocol byte. h2 no longer saw a UDP packet. The loop still ended, but after 17 passes (s1+9, s2 +8) instead of 9, because only s1 counted hops. So a half-upgraded network still forwards, but silently corrupts packets and weakens the loop protection.

b) A safer option is to give the new header its own EtherType. An old switch doesn't recognise it, stops parsing, and drops the packet. That fails safely instead of corrupting it. The rollout then goes in two steps.First, upgrade every switch to understand both the old and the new EtherType, and only then let the senders start using the new one. Until every switch is upgraded, all traffic still uses the old header, so nothing breaks. Another option is to append the new field. Old switches still read `proto_id` and `dst_id`, so they keep forwarding.

c) Data plane: the hop limit works on every packet, whatever caused the loop. That includes a manual change like `table_modify` in C1, which the controller never saw, a switch bug, or a short-lived loop while rules are being updated.`check_paths()` only knows about the rules it installed itself.

Control plane: `check_paths()` checks the rules before they are installed, so it can stop a loop before any packet is sent.

Both: The control plane prevents most mistakes in advance, and the hop limit is a safety net for the ones it misses.

d) The cost is small. Each tunnelled packet carries one extra byte, so it uses slightly more bandwidth.

## AI use

Level 3

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
