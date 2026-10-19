# OSPF Troubleshooting Lab: Hello Timer Mismatch

**Author:** Michael Bessard  
**Environment:** Cisco Packet Tracer; two Cisco 2911 routers and two PCs  
**Completed:** October 19, 2026

## Overview

I built a two-router OSPF network, verified connectivity, intentionally introduced a Hello timer mismatch, and restored service. This guided personal lab demonstrates configuration, command-output interpretation, fault isolation, and recovery verification.

## Topology and addressing

PC1 connects to R1, R1 connects to R2, and R2 connects to PC2.

| Device | Interface | IP address | Mask | Default gateway |
| --- | --- | --- | --- | --- |
| PC1 | FastEthernet0 | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| R1 | GigabitEthernet0/0 | 192.168.10.1 | 255.255.255.0 | N/A |
| R1 | GigabitEthernet0/1 | 10.0.12.1 | 255.255.255.252 | N/A |
| R2 | GigabitEthernet0/1 | 10.0.12.2 | 255.255.255.252 | N/A |
| R2 | GigabitEthernet0/0 | 192.168.20.1 | 255.255.255.0 | N/A |
| PC2 | FastEthernet0 | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |

Both routers use OSPF process 1 and area 0. R1's router ID is 1.1.1.1; R2's is 2.2.2.2. The PC-facing interfaces are passive: their networks are advertised, but they do not send OSPF Hellos.

## Working baseline

I enabled the router interfaces, assigned the addresses above, and configured OSPF with these commands.

### R1 OSPF configuration

```text
router ospf 1
 router-id 1.1.1.1
 network 10.0.12.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/0
```

### R2 OSPF configuration

```text
router ospf 1
 router-id 2.2.2.2
 network 10.0.12.0 0.0.0.3 area 0
 network 192.168.20.0 0.0.0.255 area 0
 passive-interface GigabitEthernet0/0
```

Both neighbors reached FULL. R1 learned 192.168.20.0/24 through 10.0.12.2, and R2 learned 192.168.10.0/24 through 10.0.12.1. PC-to-PC pings succeeded in both directions with 0% loss.

## Fault introduced

On R2 only, I changed the inter-router Hello interval:

```text
configure terminal
interface GigabitEthernet0/1
 ip ospf hello-interval 5
end
```

| Parameter | R1 | R2 during fault |
| --- | --- | --- |
| Hello interval | 10 seconds | 5 seconds |
| Dead interval | 40 seconds | 40 seconds |
| Interface status | up/up | up/up |

After the existing adjacency timed out, both routers logged "Dead timer expired." Neighbor tables became empty and both remote OSPF routes disappeared.

## Investigation and evidence

I compared interface status, OSPF timers, neighbor tables, routes, and connectivity:

```text
show ip interface brief
show ip ospf interface GigabitEthernet0/1
show ip ospf neighbor
show ip route ospf
```

| Test during fault | Observed result | Interpretation |
| --- | --- | --- |
| R1 ping to 10.0.12.2 | 5/5 replies | Direct IP connectivity remained available |
| PC1 ping to 192.168.20.10 | 100% loss; unreachable response from 192.168.10.1 | R1 lacked the remote LAN route |
| OSPF neighbor tables | Empty | Adjacency had been lost |
| OSPF interface timers | Hello 10 on R1; Hello 5 on R2 | Configuration mismatch identified |

The root cause was the Hello interval mismatch. The matching Dead intervals alone were insufficient to maintain adjacency.

## Resolution and validation

I restored R2's Hello interval to match R1:

```text
configure terminal
interface GigabitEthernet0/1
 ip ospf hello-interval 10
end
```

After convergence, I verified FULL adjacency on both routers, restoration of both remote LAN routes, and PC1-to-PC2 connectivity with 4/4 replies and 0% loss. I saved both router configurations using `copy running-config startup-config` and saved the resolved Packet Tracer project.

## Files and screenshots

Open [ospf-hello-timer-resolved.pkt](ospf-hello-timer-resolved.pkt) in Cisco Packet Tracer to inspect the resolved lab. To reproduce the failure, change R2's Hello interval to 5 as shown above and allow the adjacency to expire; restore it to 10 to recover.

The screenshot links below assume these filenames in the repository root.

- [Working baseline](01-baseline.png)
- [Timer mismatch and missing neighbors/routes](02-timer-mismatch.png)
- [Direct-link success and remote-PC failure](03-connectivity-failure.png)
- [Recovered adjacency, routes, and connectivity](04-recovery.png)

## MTU test observation

Before the timer exercise, I attempted `ip mtu 1400` on R2's inter-router interface and restarted OSPF. Packet Tracer accepted the command, but `show ip interface` still reported 1500 bytes on both routers, and OSPF returned to FULL. I did not reproduce an MTU mismatch failure. I removed the attempted override with `no ip mtu` before introducing the timer fault. This observation is specific to this lab; it does not establish how real IOS equipment handles the same command.

## Lessons learned

- An up/up interface and a successful direct ping do not prove that dynamic routing is healthy.
- Compare OSPF Hello and Dead intervals on both ends of a link.
- A "Dead timer expired" message describes the loss of accepted Hellos; further checks are needed to identify the cause.
- Verify a repair at three levels: adjacency, routing table, and end-to-end connectivity.
- Record observed results, including simulator behavior that differs from the intended experiment.

This repository contains a personal simulated lab and no employer configurations or operational data.
