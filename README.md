# TCP Westwood Max (`westwood_max`)

`westwood_max` is a highly aggressive throughput-optimized fork of the `westwood_sub` congestion control algorithm. It is designed specifically for erratic RF links (Wi-Fi/5G/4G) where maximizing bandwidth utilization is strictly prioritized over network fairness, latency mitigation, or standard TCP backoff compliance.

This module strips away traditional safety limits to prevent the drastic window reduction typically triggered by mobile network jitter, ACK compression, and artificial buffer saturation.

## Technical Modifications

Standard TCP variants (including BBR and original Westwood+) are inherently polite, often yielding bandwidth when facing bufferbloat. `westwood_max` employs brute-force mechanisms to maintain peak throughput:

1. **Fast Recovery:**
   Standard TCP halves the congestion window (`cwnd >> 1`) upon packet loss. `westwood_max` overrides this by dropping `ssthresh` to only 99% of the current window (`(tp->snd_cwnd * 99) / 100`). It effectively ignores random packet loss inherent to 4G/5G/Wi-Fi transitions.
2. **Window Inflation:**
   During the Congestion Avoidance phase (`TCP_CA_Open`), Reno-based algorithms increment the window linearly (+1 packet). `westwood_max` implements an unrestricted growth curve, forcibly adding +4 to `cwnd` per cycle (up to a 5000 hard-cap) to aggressively probe and saturate the link instantly.
3. **Pacing Eradication:**
   The kernel's software pacing limit (`sk_pacing_rate`) is dynamically overwritten to `~0ULL` (infinity), forcing the `fq` qdisc to flush packets as fast as the physical layer allows.
4. **ECN Ignorance:**
   Explicit Congestion Notification (CE marks) are intentionally disregarded to prevent premature throttling caused by intermediate router queue policies.

## Installation (Out-of-Tree Module)

Ensure you have your target kernel headers/source prepared.

```bash
make clean
make -C /path/to/kernel/source M=$(pwd) modules

```

Load and apply the algorithm:

```bash
insmod tcp_westwood_max.ko
sysctl -w net.ipv4.tcp_congestion_control=westwood_max

```

## Acknowledgments

This project is a direct fork of the excellent work done on [**`westwood_sub`**](https://github.com/maxsteeel/westwood-sub), which in turn traces its roots back to the original Linux 2.4 implementation of TCP Westwood+ by Angelo Dell'Aera, S. Mascolo, et al.

While the original algorithm focused on elegant end-to-end bandwidth estimation, this fork intentionally breaks those elegant rules for raw performance in modern mobile environments. Full credit for the core bandwidth estimation math remains with the original researchers and maintainers.

## Disclaimer

**Use with caution.** This is a "bully" algorithm. By deploying `westwood_max` on a shared network, this device will heavily disproportionately consume available bandwidth, potentially starving other devices on the same access point. It is not recommended for production environments where fair queueing is required.



