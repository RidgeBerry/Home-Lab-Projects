# Python Automation Project

---

<aside>
📜 Table of Contents

</aside>

# **Introduction**

Welcome to the Python Automation Project’s Documentation!

## **About**

This project serves as an introduction to my journey of learning how I can automate network configurations using Python. For this project, I did ask Gemini to give me a step-by-step guide to how to get started because I am completely new to this and my Python knowledge isn't as good as it should be. 

## Objective

To learn the basics of network automation with Python. The scripts written are going to be for pushing a simple show cdp neighbors command, and another script for setting up a basic OSPF configuration.

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
- Script 1 Workflow
    - Inventory Management: Read IPs from my_network.txt
    - Configuration Push: Uses send_command to run show cdp neighbors command
    - Logging: Prints output of the command to the terminal
- Script 2 Workflow
    - Inventory Management: Reads IPs and Config sets from device_config dictionary
    - Configuration Push: Uses send_config_set to apply OSPF on a per-interface basis
    - Logging: Prints output of the commands to the terminal with the running config

## Deployment Instructions (The “How-To”)

1. Manually configure Management IPs and SSH on all devices
2. Verify Reachability: Ping all IPs from the Laptop (192.168.1.100)
3. Execute: Run [Automation-Project-1.py](http://Automation-Project-1.py) to show cdp neighbors on all devices
4. Execute: Run [Automation-Project-2.py](http://Automation-Project-2.py) to set up OSPF on routers

## Python Scripts

Script 1:

```python
from netmiko import ConnectHandler

with open('my_network.txt') as file:    
		devices = file.read().splitlines()
		
for ip in devices: 
	  print(f"Connecting to device: {ip}")
    device_params = {
         'device_type': 'cisco_ios',
         'host': ip,
         'username': 'admin',
         'password': 'Cisco123',
     }
    try:
        with ConnectHandler(**device_params) as conn:
            #Check CDP neighbors to see if the topology is connected properly
            neighbors = conn.send_command("show cdp neighbors")
            print("--- Neighbors for {ip} ---")
            print(neighbors)
            output = conn.send_command('write')
            print(output)
    except Exception as e
         print("Error connecting to {ip}: {e}")
```

Script 2:

```python
from netmiko import ConnectHandler

device_configs = {
    '192.168.1.11': [
        'router ospf 1',
        'router-id 1.1.1.1',
        'int g0/1',
        'ip ospf 1 area 0',
        'int g0/2',
        'ip ospf 1 area 0'
    ],
    '192.168.1.12': [
        'router ospf 1',
        'router-id 2.2.2.2',
        'int g0/1',
        'ip ospf 1 area 0',
        'int g0/2',
        'ip ospf 1 area 0'
    ],
    '192.168.1.13': [
        'router ospf 1',
        'router-id 3.3.3.3',
        'int g0/1',
        'ip ospf 1 area 0',
        'int g0/2',
        'ip ospf 1 area 0'
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

SSH Key Size: Discovered that 512 bit keys default to SSHv1.5  on the routers, which is incompatible with netmiko. Corrected by changing the keys to 1024 bit
