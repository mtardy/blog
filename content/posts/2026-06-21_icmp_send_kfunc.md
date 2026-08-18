---
title: "Sending ICMP unreach control messages from BPF"
date: "2026-07-21"
categories: ["ebpf"]
tags: ["ebpf", "linux", "network"]
slug: "icmp-send-kfunc"
---

## The motivation

Tetragon started implementing network policies in a similar style as Cilium
network policy but with the extra capability of enforcing at the process level
instead of the network namespace level. One of the fundamental difference is
how Tetragon hooks the kernel to enforce the policy's rules. Tetragon loads
`cgroup_skb` BPF program types which can either return `SKB_PASS` or
`SKB_DROP` to the kernel network stack.

Thus, one UX issue arises: egress policy leading to dropping packets do not
offer any kind of feedback to the local processes trying to send traffic.
Netfilter offers a solution to this by implementing the `REJECT` verb in place
of `DROP`, which offers two things, ICMP control message and TCP reset.

This was discussed in March 2025 in Montreal at LSF/MM/BPF and summarized by an
article on lwn<span>.</span>net, [Allowing BPF programs more access to the
network](https://lwn.net/Articles/1022034/) by Daroc Alden:

> Currently, it is possible for BPF firewalls to drop packets, and therefore
> de-facto terminate a TCP connection. It would be friendlier, Tardy said in
> his second session, to send a TCP reset to immediately terminate the
> connection. This is already what other firewalls, like netfilter, do; Tardy
> wants to add a kfunc to let BPF programs do the same thing.

## Netfilter's `REJECT` target

For more details, here is an extract of the iptables man page on the reject
target:

> This is used to send back an error packet in response to the matched packet:
> otherwise it is equivalent to `DROP` so it is a terminating `TARGET`, ending
> rule traversal. This target is only valid in the `INPUT`, `FORWARD` and
> `OUTPUT` chains, and user-defined chains which are only called from those
> chains. The following option controls the nature of the error packet
> returned:
>
> `--reject-with` type given can be
> - `icmp-net-unreachable`
> - `icmp-host-unreachable`
> - `icmp-port-unreachable`
> - `icmp-proto-unreachable`
> - `icmp-net-prohibited`
> - `icmp-host-prohibited` or
> - `icmp-admin-prohibited` (using `icmp-admin-prohibited` with kernels that do
>   not support it will result in a plain `DROP` instead of `REJECT`)
>
> Which return the appropriate `ICMP` error message (`port-unreachable` is the
> default).
>
> The option `tcp-reset` can be used on rules which only match the TCP
> protocol: this causes a TCP `RST` packet to be sent back. This is mainly
> useful for blocking ident (`113/tcp`) probes which frequently occur when
> sending mail to broken mail hosts (which won't accept your mail otherwise).

## A new `bpf_icmp_send` kfunc

The proposed solution was a new kfunc to allow `cgroup_skb` to send an ICMP
control message. Note that extending the allowed returned values of
`cgroup_skb` was considered but would have been an unconventional way of
extending BPF capabilities compared to kfuncs (as well as the API guarantees)
and would not be ideal to extend to other program types.

Only ICMP send unreach was considered initially as the TCP `RST` option being a
bit more complicated to implement as a first patch set. This is often not a big
restriction as the Linux kernel network stack handles the ICMP control message
during the TCP handshake and interrupts the started connection, see:
- [`net/ipv4/tcp_ipv4.c:tcp_v4_err`](https://elixir.bootlin.com/linux/v7.2/source/net/ipv4/tcp_ipv4.c#L610-L650)
- [`net/ipv6/tcp_ipv6.c:tcp_v6_err`](https://elixir.bootlin.com/linux/v7.2/source/net/ipv6/tcp_ipv6.c#L509-L542)

Also note that the behavior is slightly different for IPv6 to read the ICMP
control message error from userspace, `IPV6_RECVERR` will need to be
explicitly set:
[`net/ipv6/datagram.c:ipv6_icmp_error`](https://elixir.bootlin.com/linux/v7.2/source/net/ipv6/datagram.c#L336-L337).
See the difference in the IPv4 and IPv6 selftests for more information.

A [first version of the patch set](https://lore.kernel.org/bpf/20250710102607.12413-1-mahe.tardy@gmail.com/)
was sent on the 10th of July 2025. After many rounds of fixes and attempts to
support additional program types (such as tc, later dropped), the series was
[finally merged upstream](https://lore.kernel.org/bpf/178367763213.156295.1145469306231918635.git-patchwork-notify@kernel.org/)
exactly one year later :birthday: after 11 revisions, on the 10th of July 2026.

Here is the [archive link on the patch set](https://lore.kernel.org/bpf/20260709144900.245904-1-mahe.tardy@gmail.com/)
that got merged upstream and here is essentially the main commit
[`f3603df9aebb ("bpf: Add bpf_icmp_send kfunc")`](https://git.kernel.org/pub/scm/linux/kernel/git/bpf/bpf-next.git/commit/?id=f3603df9aebb2a2fe2f745bd71ca38aeca60e6e7).
You can appreciate that it's mostly a call to `icmp[v6]_send` with some extra
checks to restrict its use and to avoid to recursively send control messages.

## How to use `bpf_icmp_send`

To use this new kfunc in your BPF program, first declare the signature, and in
a typical filtering program, call the function with an IPv4 or IPv6 destination
unreachable type and an appropriate unreach code.

```C
extern int bpf_icmp_send(struct __sk_buff *skb_ctx, int type, int code) __ksym __weak;

/* [...] */

SEC("cgroup_skb/egress")
int prog(struct __sk_buff *skb) {

	/* [...] */

	if (bpf_ksym_exists(bpf_icmp_send)) {
		if (skb->protocol == bpf_htons(ETH_P_IP))
			bpf_icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PKT_FILTERED);
		else if (skb->protocol == bpf_htons(ETH_P_IPV6))
			bpf_icmp_send(skb, ICMPV6_DEST_UNREACH, ICMPV6_ADM_PROHIBITED);
	}

	return SK_DROP;
}
```

Note that per the RFC, using `ICMP_PKT_FILTERED` and `ICMPV6_ADM_PROHIBITED`
seems particularly suitable for filtering BPF programs.

