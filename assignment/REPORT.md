# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): 5000

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

I started the network with my finished lab program and changed s2's tunnel entry for exit 3 (handle 2) to send packets out of port 1, back to s1. s1 already sends exit 3 towards s2, so a packet for exit 3 bounces between the two switches. I sent one packet with --tunnel 3.

In loop-before.txt, the counts went from 696 (s1) and 695 (s2) to 1334 and 1333 five seconds later. That is about 640 extra packets per switch from one packet, and the counts kept growing. s1 is always one ahead of s2, which shows it is the same packet going back and forth. The loop never ends because myTunnel_forward only sets the output port. Nothing in the packet changes between hops, so a switch cannot tell the first pass from the thousandth.

IPv4's TTL does not protect this program, for two reasons:

-Tunnelled packets never touch the TTL. In the lab they arrived with TTL 64.
-Even on the plain IPv4 path, ipv4_forward decrements the TTL but nothing ever checks it. There is no drop when it reaches 0, and v1model doesn't drop such packets by itself. Since the TTL is 8 bits, it would wrap from 0 to 255 and the packet would keep looping.

## 2. Design: the hop limit

I added an 8-bit hop_limit field to the end of the tunnel header, after dst_id.

-Size: 8 bits is the same as IPv4 TTL and IPv6 Hop Limit. That is far more than enough for this network, and it keeps the header a whole number of bytes.
-Position: I put it last so that proto_id and dst_id stay at the same positions as before, which I hoped would let old switches still read dst_id.
-Starting value: send.py starts it at 8 by default (--hops can change it). This is well above the real path length of 2 switches, but small enough that a loop only wastes a few packets.

The field is checked in the ingress, before the tunnel table:

-if it is 0, the packet is dropped;
-otherwise it is decremented by 1 and the packet is forwarded.
So it means "how many more switches this packet may cross". Checking in ingress drops a dead packet before it uses a table lookup or a queue.

The parser and deparser didn't need changing, because they follow the header definition automatically. The hosts did: send.py and receive.py had to use the same 5-byte header as the switches.

## 3. Implementation

-tunnel.p4: I added hop_limit to the header and replaced the ingress logic with the check described above.
-send.py: I added the field and a --hops option.
-receive.py: I added the field and print its value.
-tunnel_v1.p4: this is my finished lab program, kept for the mixed rollout.

My lab Steps 4–5 were not committed to main before I branched, so the tunnel.p4 diff also includes those lab changes.

While testing, I first forgot to update receive.py. Nothing showed at h2 or h3, even though the switch logs showed the tunnel table hitting correctly. receive.py was still reading a 4-byte header, so it misread the IP header and ignored the packet. This is the same lesson as lab Step 6: every device that reads the header must agree on its layout.

## 4. Evidence: it works

delivery.txt shows the packet sent with --tunnel 2 arriving at h2 and the one sent with --tunnel 3 arriving at h3. Both arrived with hop_limit 6: they started at 8, and each of the two switches decreased it by one. A plain ping h3 still works, with TTL 62.

loop-after.txt repeats the same loop in a fresh network. The counts rose by 5 on s1 and 4 on s2 (967/965 to 972/969) and then stopped. The log shows hop_limit == 0 was true exactly once, on s1. That means the packet was forwarded 8 times and dropped on the 9th switch, which matches my design.
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

## 5. Evaluation

a) Mixed rollout
Before running it, I predicted the following:

the old s2 would read only 4 tunnel bytes and so start reading IPv4 one byte too early;
it would still forward correctly by dst_id, but not decrement the hop limit, so the loop would take about twice as long;
its checksum update would write into the wrong bytes and damage the packet.
All of this happened (mixed.txt). The packet reached h2, but receive.py printed nothing. The tcpdump capture shows two things:

The hop limit arrived as 7. Only s1 had decreased it.
The IPv4 protocol byte had changed from UDP (0x11) to 0x0d. The old s2's checksum update had overwritten it. receive.py therefore no longer saw a UDP packet and ignored it.
The loop took 17 passes (9 on s1, 8 on s2) instead of 9, because only s1 was counting down. The key point is that this failure is silent: no error appears anywhere, but packets are damaged and loop protection is weaker.

b) Safe rollout
Keeping the same EtherType and just adding the field, as I did, is not safe. Old switches don't enforce the limit and corrupt the packet.

Using a new EtherType (or a version field) for the new header is safer. Old switches simply don't recognise it and drop it, so nothing gets corrupted. But tunnel traffic through old switches stops until they are upgraded.

The best approach is a staged rollout:

Upgrade every switch to a program that understands both the old and new header, while hosts keep sending the old one.
Once all switches are upgraded, switch the hosts to the new header, trying a few hosts first.
Finally, remove support for the old header.
This avoids both the corruption and the outage.

c) Data plane vs control plane
The hop limit (data plane) catches loops from any cause, while traffic is flowing. That includes manual changes like my table_modify, controller bugs, and temporary states during updates. However, it only limits the damage: the packet still loops a few times. It doesn't say where the loop is, and it needs every switch to cooperate, as C5 showed.

check_paths() (control plane) finds loops and black holes before any traffic is sent, shows exactly which entries cause them, and costs nothing per packet. But it only checks what the controller knows, so it misses manual edits and anything that changes behind its back.

Real networks use both: the control-plane check to prevent problems, and the hop limit as a safety net.

d) Cost on hardware
The change adds:

one byte to every tunnelled packet;
a few extra bits of pipeline storage (PHV) to carry the field;
a compare and a subtract in one pipeline stage, which may need an extra stage.
Hardware like PISA runs at a fixed rate, so if the program fits, there is no slowdown. If it doesn't fit, the compiler rejects it. The real cost is using up pipeline resources that other features could have used.

## AI use

I used Claude (Anthropic) during the lab and this assignment.

It explained the lab steps and logs.
It helped me debug mistakes in my parser edit, in receive.py, and in mininet commands.
It suggested the hop-limit design and code changes, which I implemented and tested.
It helped me predict and interpret the mixed-rollout result.
It drafted this report from my own evidence, which I reviewed and edited.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
