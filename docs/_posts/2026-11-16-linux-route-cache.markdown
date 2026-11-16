---
layout: post
title: "The MTU that couldn’t be: a race in Linux’s route cache"
date:   2026-11-16 18:24:49 -04:00
categories: networking
tags: [linux, bug]
---
## A brief primer on Path MTU Discovery

The maximum transmission unit (MTU) is the maximum amount of data that can be transmitted in a single packet without fragmentation. While it defaults to 1500 bytes, devices between two communicating networks may support larger or smaller MTUs than this value. Having to split data over several packets incurs overhead, so when possible it's generally preferred to use as high an MTU as possible to fit more data per packet. On the other hand, sending more data per packet than devices can receive results in the packets either outright being lost, or requiring that the data be split into several smaller packets, a process known as fragmentation. Fragmentation has the opposite effect: intermediary devices that must send to lower-MTU next-hops spend additional time splitting each packet, duplicating the IP header across each fragment wastes link bandwidth, and fragment reassembly on the end device is slow and resource-heavy.

On the Internet, the path data takes to reach its destination is unpredictable and frequently varies. As such, it's impossible to know in advance what the end-to-end MTU will be. Path MTU Discovery (PMTUD) solves this by letting senders find the largest packet size that can reach the destination without fragmentation.¹ Senders set the Don't Fragment (DF) bit on IPv4 packets (IPv6 routers never fragment at all), so when a packet is too large for the next hop, the router drops it and may respond to the sender with an ICMP message indicating this ("Fragmentation Needed" for IPv4, or ["Packet Too Big"](https://www.rfc-editor.org/rfc/rfc8201) for IPv6), including the next-hop MTU. The sender, now knowing the largest packet it can send, can cache this value for future packets sent to the destination in question.

## The problem

Within the organization I perform work for, a complaint is received about a customer's Linux host sending traffic through one of our Linux routers whose path MTU keeps being reduced to 552 bytes in 10-minute waves. Inspection of said host confirms that it's caching a discovered MTU of this value to the destination. 552 is no coincidence: Linux's `net.ipv4.route.min_pmtu` sysctl setting controls the minimum path MTU that the kernel will cache for a route, even if it receives a "Fragmentation Needed" message with lower values. *[The Internet's Path MTUs: How Big Can You Go?](https://doi.org/10.1145/3777912.3839800)* finds that 99% of IPv4 paths support at least 1420 bytes. As such, it's strikingly unusual to see a case suggesting that a device along the path can't go any higher than 552 bytes. A packet capture on the host lets us identify which device is responsible.

## Reading the captures

As some preliminary context, the packets in question are received by one of the Linux routers and forwarded on over an IPv4-in-IPv6 (`ip6tnl`) tunnel. That router's routing table also changes frequently, which will become important later.

### Who's sending these?

The capture on the host shows that every one of the "Fragmentation Needed" messages it's receiving is sourced from one of our own routers. These messages also report a next-hop MTU of 68, which is alarming as it's the legal minimum MTU for IPv4, and a sign that something is very wrong.

## Watching it happen live

Catching one of these messages on the router in the act is straightforward enough. We use tcpdump to capture ICMP packets of type 3 ("Destination Unreachable") code 4 ("Fragmentation Needed"):

{% highlight shell %}
tcpdump -i any -n -v 'icmp[0] = 3 and icmp[1] = 4'
{% endhighlight %}

{% highlight shell %}
12:47:58.048982 in0 Out IP (tos 0xc0, ttl 64, ..., proto ICMP (1), length 437)
    192.0.2.35 > 198.51.100.10: ICMP 203.0.113.117 unreachable - need to frag (mtu 68), length 417
    IP (tos 0x0, ttl 63, ..., flags [DF], proto TCP (6), length 409)
    198.51.100.10.443 > 203.0.113.117.45588: Flags [P.], ... length 357
12:48:56.471130 in0 Out ... ICMP 192.0.2.100 unreachable - need to frag (mtu 68), length 556
    IP (... ttl 63 ... flags [DF] ... length 1436) 198.51.100.10.443 > 192.0.2.100.59016: Flags [.], ... length 1384
... 3 more in the same ~30 µs, the last one length 989 [P.]
{% endhighlight %}

This confirms that the router is in fact the one sending them. Two more details in the output are worth flagging: the quoted packets carry a TTL of 63, and the second message is one of a burst of four arriving within \~30 µs.

### Where are they dropped?

The quoted packets show a TTL one lower than when the router received them. The router has therefore already forwarded² each packet once before dropping it on *output*. Had `ip_forward()`'s [own MTU check](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/ip_forward.c?h=v7.4-rc2#n135) rejected the packet, it would have done so prior to the decrement, and the quote would show the TTL as received.

### Why do they arrive in bursts?

Each message in a burst quotes a different segment of the same TCP connection, with sequence numbers running back-to-back. The router received those segments together as a single packet, merged on receive by Generic Receive Offload (GRO), so they hit the egress MTU as one. When a GRO packet is larger than the egress MTU, `ip_finish_output_gso()` [splits it back into its original segments](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/ip_output.c?h=v7.4-rc2#n275) and passes each one to `ip_fragment()`, which [sends a "Fragmentation Needed" for every segment with DF set](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/ip_output.c?h=v7.4-rc2#n588). One rejected GRO packet therefore produces one message per segment, all generated in a single pass, which is why they arrive microseconds apart.

GRO also leads to a theory for where 68 came from: perhaps the merged packets are simply too large for the egress device. The first message in the output puts that theory to rest: at 12:47:58, a single, non-GRO 409-byte packet is rejected on its own, with no burst around it. So at output time, the route's MTU really is 68, despite `ip_forward()` having checked the same packet against something larger moments earlier. PMTUD is otherwise behaving on the router; other destinations receive correct, sensible MTUs. Only traffic forwarded over the `ip6tnl` tunnel is getting 68.

### Checking the stack trace

The route itself looks perfectly sane:

{% highlight shell %}
# ip route get 192.0.2.100 from 198.51.100.10 iif in0
192.0.2.100 from 198.51.100.10 via 10.0.0.212 dev tun0
    cache expires 510sec mtu 1452 iif in0

# ip -d link show tun0
... mtu 1452 ... ip6tnl any remote ... local ... hoplimit 255 encaplimit 4 ...

# sysctl net.ipv4.route.min_pmtu net.ipv4.ip_forward_use_pmtu
net.ipv4.route.min_pmtu = 552
net.ipv4.ip_forward_use_pmtu = 0
{% endhighlight %}

With the route pointing at an `ip6tnl` device,  we propose a second theory: 68 is `IPV4_MIN_MTU`, and `ip6_tnl_xmit()` [clamps the MTU](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv6/ip6_tunnel.c?h=v7.4-rc2#n1210) it computes to that value, so maybe that clamp is somehow firing. To find out exactly where these are coming from, we can attach a probe for every “Fragmentation Needed” carrying an MTU of 68 (`htonl(68)`, which reads as `0x44000000` on a little-endian host) and print the kernel stack trace:

{% highlight shell %}
bpftrace -e 'kprobe:__icmp_send /arg1==3 && arg2==4 && arg3==0x44000000/ { @[kstack] = count(); }'
{% endhighlight %}

{% highlight shell %}
__icmp_send+1
__ip_finish_output+317        <- ip_fragment() is inlined here
ip_finish_output+41
ip_output+95
ip_forward_finish+132
ip_forward+972
ip_sublist_rcv_finish+115
ip_sublist_rcv+375
ip_list_rcv+257
...
{% endhighlight %}

This kills the `ip6tnl` clamp theory, as the packet never reaches the tunnel's transmit function at all. It dies in `__ip_finish_output()`, one of the last IPv4 steps before the device. It also leaves us with a paradox. The route's device has an MTU of 1452, and so does its cached exception. `min_pmtu` is 552, so a learned path MTU can't go below 552, let alone 68. Despite this, `ip_forward()` [checks the packet against the route MTU](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/ip_forward.c?h=v7.4-rc2#n134) and passes it, and microseconds later `ip_skb_dst_mtu()` [returns 68 for that very same packet](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/ip_output.c?h=v7.4-rc2#n307).

Both of these reads go through `ip_dst_mtu_maybe_forward()`. With `ip_forward_use_pmtu=0`, the forwarding path [ignores the cached path MTU entirely](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/net/ip.h?h=v7.4-rc2#n486) and uses the route's MTU metric (0 here), or failing that, [the MTU of the route's *device*](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/net/ip.h?h=v7.4-rc2#n499). Something has to be changing between the two reads.

## Dumping the route

Rather than continue reasoning about which code path could possibly produce 68, I hook the same `__icmp_send` call and dump everything `ip_skb_dst_mtu()` reads from the packet's route (its `dst`).

`frag_needed_dst.bt:`

{% highlight text %}
kprobe:__icmp_send /arg1 == 3 && arg2 == 4 && arg3 == 0x44000000/
{
  $skb = (struct sk_buff *)arg0;
  $dst = (struct dst_entry *)($skb->_skb_refdst & 0xfffffffffffffffe);
  $rt  = (struct rtable *)$dst;
  $sh  = (struct skb_shared_info *)($skb->head + $skb->end);
  $m   = (uint32 *)($dst->_metrics & 0xfffffffffffffffc);
  $ip  = (struct iphdr *)($skb->head + $skb->network_header);
  printf("%s dst=%s len=%u gso_size=%u\n    dev=%s dev_mtu=%u rt_pmtu=%u mtu_locked=%u uses_gw=%u rtax_mtu=%u\n",
    strftime("%H:%M:%S", nsecs), ntop($ip->daddr), $skb->len, $sh->gso_size,
    $dst->dev->name, $dst->dev->mtu, $rt->rt_pmtu, $rt->rt_mtu_locked,
    $rt->rt_uses_gateway, *($m + 1));
}
{% endhighlight %}

{% highlight shell %}
$ bpftrace frag_needed_dst.bt
13:15:03 dst=203.0.113.117 len=1436 gso_size=0
    dev=blackhole_dev dev_mtu=68 rt_pmtu=1452 mtu_locked=0 uses_gw=1 rtax_mtu=0
13:16:56 dst=192.0.2.100 len=1436 gso_size=0
    dev=blackhole_dev dev_mtu=68 rt_pmtu=1452 mtu_locked=0 uses_gw=1 rtax_mtu=0
... x13 at the same timestamp, all with an identical expiry: one route entry, one GRO train
{% endhighlight %}

There it is: `dev=blackhole_dev dev_mtu=68`. The route entry is the real one -- it still carries the exception's `rt_pmtu=1452` and `uses_gw=1` -- but in place of `tun0`, its device is `blackhole_netdev`, [whose MTU is 68](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/loopback.c?h=v7.4-rc2#n277) (`ETH_MIN_MTU`).

`blackhole_netdev` is a dummy device the kernel creates [that drops anything sent to it](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/loopback.c?h=v7.4-rc2#n246), so what’s it doing appearing in the output? Searching the source for its users turns up a handful of call sites with a common purpose. Every packet the kernel forwards carries a pointer to a route entry (a `dst_entry`) describing where it's headed and out of which device. When an entry has to be retired, typically because its device is going away, packets on other CPUs may still be holding it, so rather than free it outright, the kernel repoints it at `blackhole_netdev`. Any straggler still using the entry then sends into a sink rather than a device that may no longer exist.

The general-purpose version of this is `dst_dev_put()`. It [marks the entry as dead](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dst.c?h=v7.4-rc2#n149), [points its input and output handlers at discard functions](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dst.c?h=v7.4-rc2#n152), and repoints `dst->dev` at `blackhole_netdev` [so that the entry no longer holds a reference to the real device](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dst.c?h=v7.4-rc2#n154). The question then becomes why a route entry is being retired while a packet is still in the middle of using it.

## Reading the kernel source

The answer lives in [`net/ipv4/route.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2) and [`net/core/dst.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dst.c?h=v7.4-rc2) (as of 7.4-rc2, the last release without the fix). [Forwarded packets get their route](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1850) in `__mkroute_input()`:

{% highlight c %}
fnhe = find_exception(nhc, daddr);
if (do_cache) {
    rth = fnhe ? fnhe->fnhe_rth_input : nhc->nhc_rth_input;
    if (rt_cache_valid(rth)) {
        skb_dst_set_noref(skb, &rth->dst);   // no refcount taken
        goto out;
    }
}
// otherwise allocate a new rth, then rt_set_nexthop() -> rt_bind_exception() or rt_cache_route()
{% endhighlight %}

[Packets carry their route *without* taking a reference](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1917). This is cheap, and safe under RCU so long as nobody tears the route down underneath them. A cached route [becomes invalid](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1618) when the route generation counter (`rt_genid`) [is bumped](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n409), which happens on *any* FIB change -- and on a busy router, that is constantly. The next packet to the destination then builds a fresh route, and the old one is retired. How it's retired depends on whether the destination has a route exception (a cached per-destination path MTU entry, internally referred to as "fnhe" within the Linux kernel):

- **No exception:** `rt_cache_route()` [puts the old route on the uncached list](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1541) via `rt_add_uncached_list()` and releases it. Its device and handlers are left intact, so packets still holding it are unaffected.
- **Exception:** `rt_bind_exception()` (as well as `fnhe_flush_routes()` and `ip_del_fnhe()`) [calls `dst_dev_put()` on the old route](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1509) and then releases it. `dst_dev_put()`'s [own comment](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dst.c?h=v7.4-rc2#n141) says it's called when removing routes ahead of a device going down or being unregistered, and to make the next `dst_ops->check()` fail. Neither accounts for packets that still hold the route without a reference: here, it's retired while other CPUs may be in the middle of using it.

This is where the `blackhole_netdev` in the route dump comes from: a packet still holding a route retired via the exception path reads that route's device MTU as 68.

### The race windows

`ip_list_rcv()` [looks up routes for a whole batch of packets](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/ip_input.c?h=v7.4-rc2#n658) [before handing each one to `dst_input()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/ip_input.c?h=v7.4-rc2#n623), so what a packet experiences depends on exactly when another CPU's `dst_dev_put()` lands. Let's lay out the possibilities:

| `dst_dev_put()` lands... | Effect on the in-flight packet |
| --- | --- |
| before `dst_input()` | `dst_discard`: silent drop |
| after `dst_input()`, before `ip_forward()`'s MTU read | ICMP from `ip_forward()`, MTU 68, quoted TTL unchanged (rare: about 0.7% of the reproducer's messages) |
| after `ip_forward()`'s check, before `dst_output()` | `dst_discard_out`: silent drop |
| after `ip_output()`, before `__ip_finish_output()`'s MTU read | ICMP with MTU 68, quoted TTL decremented: what we see |

In other words, the ICMP messages the sender is receiving undercount the actual drops, as most of the windows result in the packet being silently discarded.

### Confirming the culprit

To confirm that the forwarding path is retiring routes this way, I use bpftrace again, this time counting the callers of `dst_dev_put()`:

{% highlight shell %}
bpftrace -e 'kprobe:dst_dev_put { @[kstack(5)] = count(); } interval:s:120 { exit(); }'
{% endhighlight %}

{% highlight text %}
__mkroute_input+433 <- ip_route_input_slow <- ip_route_input_noref <- ip_rcv_finish_core : 2918
...
{% endhighlight %}

This works out to roughly 24 retirements per second on the forwarding path alone. The callees are inlined, but in 7.4-rc2 the only `dst_dev_put()` callers reachable from within `__mkroute_input()` are `rt_bind_exception()`, `fnhe_flush_routes()` and `ip_del_fnhe()`. All three of these exist solely for routes with an exception. Yet of those roughly 1,400 retirements a minute, only a handful produce an ICMP message, because a retirement only does harm if another CPU happens to be using the old route at that exact moment. Plenty of opportunities and very few hits smells exactly like a narrow race.

## Why did every destination have an exception?

Route exceptions normally come from genuine path MTU discovery, and should be rare. Yet `ip route get` shows `cache expires ... mtu 1452` on every destination routed over the tunnel, so something is creating them en masse. Let's check who, again with bpftrace:

{% highlight shell %}
bpftrace -e 'kprobe:ip_rt_update_pmtu { @[kstack(4), arg3] = count(); } interval:s:30 { exit(); }'
{% endhighlight %}

| Caller | MTU reported | Count (30 s) | Tunnel |
| --- | --- | --- | --- |
| `ipxip6_tnl_xmit` | 1452 | 796,692 (\~26.5k/s) | device MTU 1452, `encaplimit 4` |
| `ipxip6_tnl_xmit` | 1460 | 1,048 | device MTU 1452, `encaplimit none` |

On every packet it transmits, `ip6_tnl_xmit()` [computes the inner MTU the tunnel can carry and reports it to the inner IPv4 route](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv6/ip6_tunnel.c?h=v7.4-rc2#n1205):

{% highlight text %}
mtu = outer_mtu - 40 (IPv6 header) - 8 (encap-limit destination option, unless encaplimit none)
skb_dst_update_pmtu_no_confirm(skb, mtu);
{% endhighlight %}

The tunnel in question was created without `encaplimit none`, and so has [iproute2's default of](https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/tree/ip/link_ip6tnl.c#n82) `encaplimit 4`. Its device MTU is [set to the underlay MTU minus 48](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv6/ip6_tunnel.c?h=v7.4-rc2#n1524) to account for both headers, so the tunnel reports 1500 - 40 - 8 = 1452 -- exactly the device MTU. Over in [`__ip_rt_update_pmtu()`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1060):

{% highlight c %}
if (old_mtu < mtu)
    return;        // only a strictly larger report is ignored
{% endhighlight %}

A report *equal* to the current MTU isn't ignored. For a destination without an exception, [it creates one](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1094) saying "MTU 1452"; once one exists, the steady stream of reports (roughly 26,500 a second across the tunnel) [refreshes it whenever it's past half its lifetime](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1070), so it never expires. This changes nothing about the size of any packet, but it puts every destination behind these tunnels on the exception path, where old routes are retired with `dst_dev_put()`.

Conveniently, the router's own configuration serves as a natural control. Another `ip6tnl` tunnel on the same router runs the exact same code with a device MTU of 1452, but was created with `encaplimit none`. It reports 1460, which is larger than the device MTU, so [the kernel returns early](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv4/route.c?h=v7.4-rc2#n1060): no exception is created, and no route is ever blackholed.

## Reproducing it

With the mechanism understood, we can attempt a reproducer. For this, we need four components: a sender, router, tunnel peer and receiver. The router forwards over an `ip6tnl` tunnel with the default `encaplimit` and an MTU of underlay - 48, which creates the exception. Sustained multi-flow traffic is pushed through the router from the sender while a tight `ip route add`/`ip route del` loop bumps the route generation counter, and `tcpdump 'icmp[0]=3 and icmp[1]=4 and icmp[6:2]=68'` on the sender watches for the bogus messages. The same setup with `encaplimit none` serves as the control.

### Setup

{% highlight text %}
 netns A (sender)            root netns = R (router)                          netns B (receiver)
 10.1.0.2/24  a0 ──veth── rA 10.1.0.1/24
                              tnl 10.200.0.1/30  ══ ip6tnl (mtu 1452) ══  tnl 10.200.0.2/30
                              rB fd00::1 ──veth (mtu 1500)──────────────  b0 fd00::2
                              route 10.99.0.0/24 via 10.200.0.2 dev tnl    lo 10.99.0.1/32 (iperf3 -s)
{% endhighlight %}

We use `iperf3 -P 8` to generate traffic between A and B with 8 parallel streams so several of R's CPUs forward simultaneously. A loop adding and deleting a /32 on R's loopback causes the route generation counter to be repeatedly bumped:

{% highlight shell %}
yes $'route add 198.51.100.1/32 dev lo\nroute del 198.51.100.1/32 dev lo' | ip -force -batch -
{% endhighlight %}

One subtlety: R has to be the root network namespace. `__icmp_send()` takes its namespace from the route's device, and `blackhole_netdev` lives in the initial namespace, so if R ran in a child namespace, its messages would be sent from the root namespace instead and never reach the sender.

### Results

To count every hit, a netfilter rule on the sender drops the "Fragmentation Needed" messages so it doesn't act on any of them, while tcpdump on the sender counts them as they arrive. Over 60-second runs on unpatched 7.4-rc1 kernels:

| vCPUs | Tunnel | FIB churn | MTU 68 messages | `dst_dev_put()` calls | TCP retransmits |
| --- | --- | --- | --- | --- | --- |
| 4 | `encaplimit 4` (exception) | yes | 40,185 | 496,789 | 345,162 |
| 4 | `encaplimit none` (no exception) | yes | 0 | 0 | 12,421 |
| 4 | `encaplimit 4` (exception) | no | 0 | 0 | 10,522 |
| 8 | `encaplimit 4` (exception) | yes | 505,291 | 2,053,315 | 3,877,208 |
| 8 | `encaplimit none` (no exception) | yes | 0 | 0 | 14,226 |
| 8 | `encaplimit 4` (exception) | no | 0 | 0 | 177 |

The bug needs both conditions. An exception without churn, or churn without an exception, produces no bogus messages and a normal retransmit count. With both, retransmits climb to between roughly 28 and 270 times those of the encaplimit none runs. Going from 4 to 8 vCPUs raises the message rate from about 670 to about 8,400 a second, roughly twelvefold, as you'd expect from a race between CPUs. In every run, every bogus message came from a route on `blackhole_dev` still carrying the exception's MTU of 1452.&#32;

## Working around it

Until a fix makes it into the kernels we run, the goal is to stop the tunnel from creating exceptions in the first place. What matters is how the MTU the tunnel reports compares to its device MTU:

| Setup | Device MTU | Tunnel reports | Result |
| --- | --- | --- | --- |
| Original (`encaplimit 4`) | 1452 | 1452 | Equal: exception created, exposed |
| `encaplimit none` | 1452 | 1460 | Report higher: ignored, safe |
| `encaplimit 4`, MTU lowered to 1444 | 1444 | 1452 | Report higher: ignored, safe |

The original MTU isn't too high; 1452 is exactly right for `encaplimit 4`. The problem is purely the equality. Both `encaplimit none` and a lower MTU avoid it by the same mechanism, but I opt for the former. It leaves the device MTU untouched rather than giving up 8 bytes of genuine capacity, drops the 8-byte option header from every packet instead, and is the variant the reproducer confirmed. The only thing given up is [RFC 2473](https://www.rfc-editor.org/rfc/rfc2473)'s guard against nested encapsulation loops.

Notably, the `- 48` stays. With `encaplimit none` the tunnel overhead is only 40 bytes, but the extra 8 bytes are now precisely what keeps the device MTU below the reported MTU. Changing it to `- 40` would recreate the equality, and the bug along with it.

### Applying it live

Changing the encaplimit of a live tunnel has one trap, however. `ip6_tnl_change()` [calls](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv6/ip6_tunnel.c?h=v7.4-rc2#n1557) `ip6_tnl_link_config()`, which [*recomputes* `dev->mtu` from the underlay](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv6/ip6_tunnel.c?h=v7.4-rc2#n1524). With `encaplimit none`, [that's underlay - 40](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/ipv6/ip6_tunnel.c?h=v7.4-rc2#n1525), or 1460. That's the computed MTU again, which brings the equality, and the bug, straight back. The fix is to set the MTU in the same netlink request, which [applies the tunnel change first and the MTU second](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/rtnetlink.c?h=v7.4-rc2#n3902) under a single lock:

{% highlight shell %}
ip link set dev "$dev" mtu "$cur_mtu" type ip6tnl encaplimit none
{% endhighlight %}

## Fixing it upstream

The race isn't new. The exception paths have had it since [95c47f9cf5e0](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=95c47f9cf5e0) in 2017, and a 2019 fix, [5018c59607a5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5018c59607a5), addressed the same problem for `rt_cache_route()` only.

I've since [submitted an upstream patch](https://lore.kernel.org/netdev/20261105182640.2999521-1-me@chasehil.tz/) which has been [merged into mainline](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ae89f2724db1afa96a1f1b661174fc41ae1e88fa), landing in 7.4-rc3.

### Notes

¹ In practice, PMTUD is widely broken. Many firewalls and router filter or rate-limit ICMP, causing senders' packets to simply be lost. For this reason,  [Packetization Layer PMTUD](https://www.rfc-editor.org/rfc/rfc4821) was designed to work without ICMP.

² Here, "forwarded" means the kernel has routed the packet from one net device towards another, not that it has left the system.
