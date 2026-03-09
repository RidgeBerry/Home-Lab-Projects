# Structured Data Parsing & Multi-Threading

---

<aside>
📜 **TABLE OF CONTENTS**

</aside>

# **Introduction**

This project is really about two things: learning how to pull useful data out of a switch without doing it manually, and making the whole process faster. I started by writing a script that uses **Regular Expressions** to hunt through "show" commands for specific errors, but I quickly realized how easily those can break if the output changes even a little bit. To fix that, I moved over to using the **Cisco Genie Library**, which turns that messy text into a clean format that Python can actually understand. I also added **Threading** so the script can talk to multiple switches at the same time instead of waiting on them one by one.

## Network Topology

![Mega Lab topology.png](Mega_Lab_topology.png)

### Technical Stack

- **Hardware:** 3x Cisco 2921 Routers, 3x Cisco Catalyst 2960 Switches.
- **Automation:** Python 3.13.11, Netmiko (SSH-based management), Cisco Genie (pyATS).
- **Concurrency:** Python `concurrent.futures` (ThreadPoolExecutor).
- Data Formats: JSON, Structured Text

### Key Features

- **Structured Parsing:** Utilizes Genie to convert raw CLI "show" commands into Python dictionaries.
- **Parallel Execution:** Multi-threaded connectivity allowing up to 10 simultaneous SSH sessions.
- **Health Auditing:** Automated detection of CRC errors, interface status, and CPU thresholds.
- **Drift Remediation:** Built-in configuration sets to reset CDP and Vlan1 status across the fleet.

## Automation Scripts

1. Manual Regex Troubleshooting 
    
    This Script uses standard Regular Expressions to hunt for patterns in raw CLI strings. It represents the beginner approach to automating basic troubleshooting
    
    ```python
    from netmiko import ConnectHandler
    import re
    
    with open('my_network.txt') as file:
        devices = file.read().splitlines()
     
    intf_pattern = re.compile(r"^(\S+) is up, line protocol is up", re.IGNORECASE)
    crc_pattern = re.compile(r"(\d+) CRC", re.IGNORECASE)
    err_pattern = re.compile(r"(\d+) input errors", re.IGNORECASE)
    cpu_pattern = re.compile(r"five minutes:\s+(\d+)%", re.IGNORECASE)
    mac_pattern = re.compile(r"Total Mac Addresses.*:\s+(\d+)", re.IGNORECASE)
    ospf_pattern = re.compile(r"(\d+\.\d+\.\d+\.\d+)\s+\d+\s+(\w+)/", re.IGNORECASE)
    duplex_pattern = re.compile(r"([a-zA-Z]+)-duplex")
    speed_pattern = re.compile(r",\s+([\w./]+)Mb/s")
    count = 0
    
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
               print(f"--- Device: {ip} ---")
               raw_output = conn.send_command("show interfaces")
               interface_blocks = re.split(r"\n(?=[A-Za-z]+[\d/]+ is)", raw_output)
               print(f"{'Interface':<30} | {'Status':<10} | {'CRC':<5} | {'Errors':<5} | {'Duplex':<5} | {'Speed':<5}")
               print("-" * 70)
               for block in interface_blocks:
                    count +=1
                    name_match = intf_pattern.search(block)
                    if name_match:
                        interface_name = name_match.group(1)
                        crc_match = crc_pattern.search(block)
                        err_match = err_pattern.search(block)
                        crc_val = int(crc_match.group(1)) if crc_match else 0
                        err_val = int(err_match.group(1)) if err_match else 0
                        status_label = "HEALTHY"
                        if crc_val > 0 or err_val > 0:
                            status_label = "!! BAD !!"
                        duplex_match = re.search(r"(\w+)-duplex,\s+([\w./]+)", block)
                        if duplex_match:
                            duplex_label = duplex_match.group(1)
                            speed_label = duplex_match.group(2)
                        else:
                            duplex_label = 'N/A'
                            speed_label = 'N/A'
                        print(f"{interface_name:<30} | {status_label:<10} | {crc_val:<5} | {err_val:<5} | {duplex_label:<5} | {speed_label:<5}")
                    count = 0
               raw_output = conn.send_command("show processes cpu sorted | exclude 0.00%")
               cpu_match = cpu_pattern.search(raw_output)
               if cpu_match:
                   cpu_val = int(cpu_match.group(1))
                   if cpu_val > 80:
                       print(f"!! HIGH CPU: {cpu_val}% !!")
                   else:
                       print(f"CPU usage is Good it is at {cpu_val}")
               raw_output = conn.send_command("show mac address-table count")
               mac_match = mac_pattern.search(raw_output)
               if mac_match:
                   mac_val = int(mac_match.group(1))
                   if mac_val > 1000:
                       print(f"!! HIGH MAC COUNT: {mac_val} !!")
               raw_output = conn.send_command("show ip ospf neighbor")
               ospf_match = ospf_pattern.findall(raw_output)
               print(ospf_match)
               if ospf_match:
                   print(f"Found {len(ospf_match)} OSPF neighbors on {ip}")
                   for router_id, state in ospf_match:
                       if state.lower() != "full":
                           print(f"!! OSPF Neighbor {router_id} in {state} state !!")
                       else:
                           print(f"!! OSPF Neighbor {router_id} in {state} state !!")
    
        except Exception as e:
            print(f"Error connecting to {ip}: {e}")
    ```
    

1. Multi-Threaded Genie Parser
    
    This is the more refined version of the tool. It moves away from sequential processing and utilizes Genie to handle data objects rather than text blocks.
    
    ```python
    import json
    from netmiko import ConnectHandler
    from concurrent.futures import ThreadPoolExecutor
    
    def connect_to_device(ip):
        device_params = {
            'device_type': 'cisco_ios',
            'host': ip,
            'username': 'admin',
            'password': 'Cisco123',
        }
    
        try:
            with ConnectHandler(**device_params) as conn:
                print(f"Successfully connected to {ip}")
                output_file = f"information_for_{device_params['host']}.txt"
    
                version_output = conn.send_command("show version", use_genie=True)
                # version_json = f"version_{device_params['host']}.json"
                # with open(version_json, 'w') as f:
                #     json.dump(version_output, f, indent=4)
                
                interfaces_output = conn.send_command("show interfaces", use_genie=True)
    
                # filename = f"inventory_{device_params['host']}.json"
                # with open(filename, 'w') as f:
                #     json.dump(interfaces_output, f, indent=4)
    
                cpu_output = conn.send_command("show processes cpu sorted | exclude 0.00%", use_genie=True)
                # cpu_json = f"cpu_information__{device_params['host']}.json"
                # with open(cpu_json, 'w') as f:
                #     json.dump(cpu_output, f, indent=4)
    
                cdp_output = conn.send_command("show cdp neighbors", use_genie=True)
                # cdp_json = f"cdp_information__{device_params['host']}.json"
                # with open(cdp_json, 'w') as f:
                #     json.dump(cdp_output, f, indent=4)
    
                cdp_update = conn.send_config_set(["int vlan 1", "shutdown" ])
                cdp_update = conn.send_command("no cdp run")
                cdp_update = conn.send_command("cdp run")
                
    
                with open(output_file, 'w') as file:
                    print(f"Information for {ip}", file= file)
                    print("-" * 80, file=file)
                    version_dictionary = version_output["version"]
                    hostname = version_dictionary["hostname"]
                    platform = version_dictionary["platform"]
                    version = version_dictionary["version"]
                    uptime = version_dictionary["uptime"]
                    print(f"Hostname: {hostname}", file= file)
                    print(f"Platform: {platform}", file= file)
                    print(f"Version: {version}", file= file)
                    print(f"Uptime: {uptime}", file= file)
    
                    print(f"\n\n", file= file)
    
                    print(f"{'Interface':<30} | {'Status':<10} | {'CRC':<5} | {'Errors':<5} | {'Duplex':<5} | {'Speed':<5}", file=file)
                    print("-" * 80, file=file)
                    for interface, data in interfaces_output.items():
                        status = data.get("oper_status", "N/a")
                        counters = data.get("counters", {})
                        crc = counters.get("in_crc_errors", "N/a")
                        errors = counters.get("in_errors", "N/a")
                        duplex = data.get("duplex_mode", 'N/a')
                        speed = data.get("port_speed", "N/a")
                        print(f"{interface:<30} | {status:<10} | {crc:<5} | {errors:<5} | {duplex:<5} | {speed:<5}", file=file)
    
                    print(f"\n\n", file=file)
                    print(f"CPU Five Minute Usage", file=file)
                    print("-" * 80, file=file)
                    cpu_usage = cpu_output["five_min_cpu"] 
                    print(f"{cpu_usage}", file=file)
    
                    print(f"\n\n", file=file)
                    print(f"CDP Neighbors", file=file)
                    print("-" * 80, file=file)
                    cdp_neighbors = cdp_output
                    print(f"{cdp_neighbors}", file=file)
    
        except Exception as e:
            print(f"Failed to connect to {ip}: {e}")
    
    def main():
        with open("my_network.txt") as file:
            devices = file.read().splitlines()
    
        with ThreadPoolExecutor(max_workers=10) as executor:
            executor.map(connect_to_device, devices)
    
    if __name__ == "__main__":
        main()
    ```
    

## Lessons Learned & Technical Challenges

### 1. The "Data Integrity" Challenge

**Challenge:** When using Regex, if a switch had a slightly different software version, the string "Total Mac Addresses" might change to "Total MAC addresses," causing my script to return `None`.
**Solution:** Implementing Genie moved the responsibility of parsing to a maintained Cisco library, ensuring my scripts work across different IOS versions without manual pattern updates.

### 2. Thread-Safety in Reporting

**Challenge:** In my first attempt at multi-threading, all 10 threads tried to write to the terminal at once, resulting in scrambled text.
**Solution:** I refactored the script to create unique individual files (e.g., `information_for_192.168.99.11.txt`) for each device. This prevented "race conditions" and created a clean audit trail.

### 3. The Dictionary Key Error

**Challenge:** Some interfaces didn't have "counters" (like Null0). The script would crash with a `KeyError`.
**Solution:** I learned to use the `.get()` method in Python dictionaries (e.g., `data.get("counters", {})`). This allows the script to fail gracefully by returning an empty dictionary instead of crashing.

---