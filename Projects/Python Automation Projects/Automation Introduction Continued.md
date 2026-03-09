# Python Automation Project Continued

---

<aside>
📜 Table of Contents

</aside>

# **Introduction**

Welcome to the Python Automation Project Continued Documentation!

## **About**

This project is a continuance of my last Python project. It will serve as a place for me to document the scripts that I continue to make to practice network automation with python.

## Objective

To practice writing scripts for network automation

## Network Topology

Hardware: 3x Cisco 2921 Routers, 3x Cisco Catalyst 2960 Switches

Physical Connections:

- Management Hub: Switch SW1
    - Ports: Fa0/1-4, g0/1-2
- Automation Host: Laptop connected to SW1, port fa0/1
- Links:
    - SW1 (g0/1) ←→ R1 (g0/0)
    - SW1 (g0/2) ←→ R2 (g0/0)
    - SW1 (fa0/2) ←→ R3 (g0/0)
    - SW1 (fa0/3) ←→ SW2 (fa0/24)
    - SW1 (fa0/4) ←→ SW3 (fa0/24)
    - R1 (g0/1) ←→ R2 (g0/1)
    - R1 (g0/2) ←→ R3 (g0/1)
    - R2 (g0/2) ←→ R3 (g0/2)

## Inventory & IP Schema

| Device | Management IP | Role | OSPF ID |
| --- | --- | --- | --- |
| R1 | 192.168.1.11 | Core Router | 1.1.1.1 |
| R2 | 192.168.1.12 | Core Router | 2.2.2.2 |
| R3 | 192.168.1.13 | Core Router | 3.3.3.3 |
| SW1 | 192.168.1.21 | Management Hub | N/a |
| SW2 | 192.168.1.22 | Access Switch | N/a |
| SW3 | 192.168.1.23 | Access Switch | N/a |

## Automation Components

- Prerequisites
    - Python Version: 3.10+
    - Libraries: Netmiko
    - Security: SSHv2 enabled with 1024-bit RSA keys

## Deployment Instructions (The “How-To”)

1. Manually configure Management IPs and SSH on all devices
2. Verify Reachability: Ping all IPs from the Laptop (192.168.1.100)
3. Execute: Run the scripts 

## Python Scripts

Script 1: VTP

```python
from netmiko import ConnectHandler

device_config = {
    '192.168.1.21': [
        'vtp mode server',
        'vtp domain CCNA',
        'vlan 10',
        'name Research',
        'vlan 20',
        'name Accounting',
        'vlan 99',
        'name Management',
    ],
    '192.168.1.22': [
        'vtp mode client',
        'vtp domain CCNA',
    ],
    '192.168.1.23': [
        'vtp mode client',
        'vtp domain CCNA',
    ]
}

for ip, commands in device_config.items():
    print(f"Connecting to device: {ip}")
    device_params = {
        'device_type': 'cisco_ios',
        'host': ip,
        'username': 'admin',
        'password': 'Cisco123',
    }

    try:
        with ConnectHandler(**device_params) as conn:
           output = conn.send_config_set(commands)
           print(output)
           print("--- VTP Status ---")
           vtp_status = conn.send_command('show vtp status')
           print(vtp_status)
           print("--- VLANS ---")
           vlans = conn.send_command('show vlan')
           print(vlans)

    except Exception as e:
        print("Error connecting to {ip}: {e}")
```

Script 2: EtherChannel

```python
from netmiko import ConnectHandler

device_config = {
    '192.168.1.22': [
        'int range fa0/1-2',
        'channel-protocol lacp',
        'channel-group 1 mode active',
        'int port-channel 1',
        'switchport mode trunk',
        'switchport trunk allowed vlan 10,20,99',
        'switchport trunk native vlan 99'
    ],
    '192.168.1.23': [
        'int range fa0/1-2',
        'channel-protocol lacp',
        'channel-group 1 mode active',
        'int port-channel 1',
        'switchport mode trunk',
        'switchport trunk allowed vlan 10,20,99',
        'switchport trunk native vlan 99'
    ]
}

for ip, commands in device_config.items():
    print(f"Connecting to device: {ip}")
    device_params = {
        'device_type': 'cisco_ios',
        'host': ip,
        'username': 'admin',
        'password': 'Cisco123',
    }

    try:
        with ConnectHandler(**device_params) as conn:
           output = conn.send_config_set(commands)
           print(output)
           print("--- Etherchannel Summary ---")
           etherchannel_summary = conn.send_command('show etherchannel summary')
           print(etherchannel_summary)
           print("--- Trunk Interfaces ---")
           trunk = conn.send_command('show int trunk')
           print(trunk)

    except Exception as e:
        print("Error connecting to {ip}: {e}")
```

## Script 3: DHCP Config

```python
from netmiko import ConnectHandler

device_configs = {
    '192.168.1.11': [
        'ip dhcp excluded-address 192.168.1.1 192.168.1.30',
        'ip dhcp pool Pool1',
        'network 192.168.1.0 255.255.255.0',
        'dns-server 8.8.8.8',
    ],
}

for ip, commands in device_configs.items():
    print(f"Connecting to device: {ip}")
    device_params = {
        'device_type': 'cisco_ios',
        'host': ip,
        'username': 'admin',
        'password': 'Cisco123',
    }

    try:
        with ConnectHandler(**device_params) as conn:
            output = conn.send_config_set(commands)
            print(output)
            run_config = conn.send_command('sh run')
            print(f"--- Running Config for {ip} ---")
            print(run_config)
        
    except Exception as e:
        print(f"Error connecting to {ip}: {e}")
```

## Known Issues & Lessons Learned

I haven't had any issues with the scripts so far.
