# CCNA Command Cheat Sheet

> One growing reference for every config command. Add to it as you go.
> This is your fast lookup during labs and your last-week cram sheet.
> Convention: `!` lines are comments/notes, everything else is real syntax.

---

## Basic device setup / CLI

```
enable
configure terminal
hostname R1
no ip domain-lookup          ! stop DNS lookup on typos
```

```
! verify:
show running-config
show version
```

## Device security

```
! console + vty + enable, SSH — fill in as you learn them
```

## Interfaces

```
interface g0/0
  ip address 192.168.1.1 255.255.255.0
  no shutdown
```
```
! verify:
show ip interface brief
show interfaces status
```

## VLANs & trunking

```
```

## STP / RSTP

```
```

## EtherChannel

```
```

## Static routing

```
ip route 10.0.0.0 255.255.255.0 192.168.1.2
```

## Dynamic routing — RIP / EIGRP

```
```

## OSPF

```
```

## First Hop Redundancy (HSRP)

```
```

## IPv6

```
```

## ACLs (standard + extended)

```
```

## NAT

```
```

## Services — DHCP / DNS / NTP / SNMP / Syslog

```
```

## Remote access — SSH / FTP / TFTP

```
```

## Security hardening — Port Security / DHCP Snooping / DAI

```
```

## QoS

```
```

## Wireless (WLC)

```
```

---

## Show / debug quick reference

<!-- The commands you reach for constantly when troubleshooting. -->

```
show ip route
show ip interface brief
show mac address-table
show vlan brief
show cdp neighbors
```
