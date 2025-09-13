---
title: ARP Spoofing & IP Source Guard
date: 2022-07-13 14:20:00 +0800
categories: [Network_Security]
tags: [ARP, spoofing, mitigation, switching, security]
---

## Introduction

ARP is one of the most important networking protocols that other protocols rely on as it maps a MAC address to an associated IP address. The attack we will be talking about is called ARP Spoofing, which exploits the inherent trust in the ARP protocol.

## Understanding ARP Protocol

The Address Resolution Protocol (ARP) operates at Layer 2 and is responsible for:

- **MAC to IP mapping**: Resolves IP addresses to MAC addresses
- **Cache maintenance**: Maintains ARP table entries
- **Broadcast resolution**: Uses broadcast to discover unknown mappings

### ARP Table Example

```bash
# View current ARP table
arp -a

Interface: 192.168.1.100 --- 0x2
  Internet Address      Physical Address      Type
  192.168.1.1           aa-bb-cc-dd-ee-ff     dynamic
  192.168.1.50          11-22-33-44-55-66     dynamic
```

## ARP Spoofing Attack

### Attack Mechanism

ARP spoofing works by:

1. **Monitoring traffic**: Attacker listens for ARP requests
2. **Sending fake replies**: Responds with malicious MAC address
3. **Cache poisoning**: Corrupts victim's ARP table
4. **Traffic interception**: Redirects traffic through attacker

### Attack Example

```python
from scapy.all import *

def arp_spoof(target_ip, gateway_ip, interface):
    # Get MAC addresses
    target_mac = get_mac(target_ip)
    gateway_mac = get_mac(gateway_ip)
    
    # Create ARP responses
    packet1 = ARP(op=2, pdst=target_ip, hwdst=target_mac, 
                  psrc=gateway_ip)
    packet2 = ARP(op=2, pdst=gateway_ip, hwdst=gateway_mac, 
                  psrc=target_ip)
    
    # Send packets
    send(packet1, iface=interface)
    send(packet2, iface=interface)
```

## IP Source Guard Mitigation

### Configuration on Cisco Switches

```cisco
! Enable IP Source Guard on interface
interface FastEthernet0/1
 ip verify source
 ip verify source mac-address
 
! Configure DHCP snooping (required for IP Source Guard)
ip dhcp snooping
ip dhcp snooping vlan 10
ip dhcp snooping trust

! Configure trusted interfaces
interface GigabitEthernet0/1
 ip dhcp snooping trust
```

### Dynamic ARP Inspection (DAI)

```cisco
! Enable Dynamic ARP Inspection
ip arp inspection vlan 10

! Configure trusted interfaces
interface GigabitEthernet0/1
 ip arp inspection trust
 
! Set rate limits
ip arp inspection limit rate 15
```

## Detection Methods

### Network Monitoring

```bash
# Monitor ARP traffic with tcpdump
sudo tcpdump -i eth0 arp

# Look for suspicious patterns:
# - Multiple MAC addresses for same IP
# - High volume of ARP replies
# - Gratuitous ARP from unexpected sources
```

### SNMP Monitoring

```python
from pysnmp.hlapi import *

def monitor_arp_table(switch_ip, community):
    for (errorIndication, errorStatus, errorIndex, varBinds) in nextCmd(
        SnmpEngine(),
        CommunityData(community),
        UdpTransportTarget((switch_ip, 161)),
        ContextData(),
        ObjectType(ObjectIdentity('1.3.6.1.2.1.4.22.1.2')),
        lexicographicMode=False):
        
        if errorIndication:
            print(errorIndication)
            break
            
        for varBind in varBinds:
            print(' = '.join([x.prettyPrint() for x in varBind]))
```

## Best Practices

### Network Segmentation

1. **VLAN isolation**: Separate critical systems
2. **Access control**: Implement port security
3. **Traffic filtering**: Use ACLs to control ARP traffic

### Monitoring and Alerting

```bash
#!/bin/bash
# ARP monitoring script
while true; do
    # Check for duplicate IP addresses
    arp -a | awk '{print $2}' | sort | uniq -d | while read ip; do
        echo "ALERT: Duplicate IP detected: $ip"
        # Send alert notification
        mail -s "ARP Anomaly Detected" security@company.com
    done
    sleep 30
done
```

## Conclusion

ARP spoofing remains a significant threat in modern networks. Implementing proper mitigation techniques such as:

- **IP Source Guard**
- **Dynamic ARP Inspection** 
- **DHCP Snooping**
- **Network monitoring**

These controls provide layered defense against ARP-based attacks and help maintain network integrity.

Remember that security is an ongoing process, and regular monitoring of ARP tables and network traffic is essential for early detection of malicious activities.