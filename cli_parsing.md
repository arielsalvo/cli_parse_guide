## Parsing semi-structured text with Ansible

As the scope of automation increases, it's not uncommon to encounter a device, host or platform that only supports a command-line interface and  commands issued returns semi-structured text.

With the 1.2.0 release of the ansible.netcommon collection, Ansible now has available a single module to both run commands and parse the output.

In addition to the native parsing engine, the cli_parse module works with several third party parsing engines.

## Why parse the text?

Some of the more common use cases for parsing semi-structured text into Ansible native data stuctures include:

- Used with the `when` clause to conditionally run other tasks or roles
- Configuration and operational state compliance checking used with the `assert` module
- Used with the `template` module to generate reports about configuration and operational state information
- Generate device commands or configuration when used with templates and `command` of `config` modules
- Suppliment native facts information when used along side current platform `facts` modules
- Populate the Ansible inventory or a CMDB with new structured data

## Skip the parsing

- When the device, host or platform has a RESTAPI and returns json
- Exisitng Ansible facts modules already return the desired data
- Ansible network resource modules exist for configuration mangement of the device and resource

## Design overview

`cli_parse` in an ansible module that can either run a cli command on a device and return a parsed result or can simply parse any text document.

`cli_parse` includes cli_parser plugins to interface with with a variety of parsing engines.  At the time of release, cli_parsing plugins are included for the following parsing engines:

- `native`:  The native parsing engine is built into ansible and requires no addition python libraries
- `xml`: Convert XML to an ansible native data structure
- `textfsm`: A python module which implements a template based state machine for parsing semi-formatted text
- `ntc_templates`: Predefined textfsm templates packages supporting a variety of platforms and commands
- `ttp`: A library for semi-structured text parsing using templates, with added capabilities to simplify the process
- `pyats`: Use the parsers included with Cisco's Test Automation & Validation Solution
- `json`: Convert json output at the cli to an ansible native data structure

Because cli_parse uses a plugin based architecture additional parsing engines can be used from any ansible collection.

## Parsing engine specifics

### Native parsing engine

The native parsing engine is included with the cli_parse module. It uses data captured using regular expressions to populate the parsed data structure.

The native parsing engine requires a template be used to parse the command output. A native template stored as a yaml file.


#### Networking example

Example output of the `show interface` command:

```
Ethernet1/1 is up
admin state is up, Dedicated Interface
  Hardware: 100/1000/10000 Ethernet, address: 5254.005a.f8bd (bia 5254.005a.f8bd)
  MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec
  reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, medium is broadcast
  Port mode is access
  full-duplex, auto-speed
  Beacon is turned off
  Auto-Negotiation is turned on  FEC mode is Auto
  Input flow-control is off, output flow-control is off
  Auto-mdix is turned off
  Switchport monitor is off
  EtherType is 0x8100
  EEE (efficient-ethernet) : n/a
  Last link flapped 4week(s) 6day(s)
  Last clearing of "show interface" counters never
<...>
```

An example template stored as `templates/nxos_show_interface.yaml`

```yaml
---
- example: Ethernet1/1 is up
  getval: '(?P<name>\S+) is (?P<oper_state>\S+)'
  result:
    "{{ name }}":
      name: "{{ name }}"
      state:
        operating: "{{ oper_state }}"
  shared: true

- example: admin state is up, Dedicated Interface
  getval: 'admin state is (?P<admin_state>\S+),'
  result:
    "{{ name }}":
      name: "{{ name }}"
      state:
        admin: "{{ admin_state }}"

- example: "  Hardware: Ethernet, address: 5254.005a.f8b5 (bia 5254.005a.f8b5)"
  getval: '\s+Hardware: (?P<hardware>.*), address: (?P<mac>\S+)'
  result:
    "{{ name }}":
      hardware: "{{ hardware }}"
      mac_address: "{{ mac }}"
```

A native parser template is stuctured as a list of parsers, each containing the following key, value pairs:

- `example`: An example line of the text line to be parsed
- `getval`: A regular expression using named capture groups to store the extracted data
- `result`: A data tree, populated as a template, from the parsed data
- `shared`: (optional) The shared key makes the parsed values available to the rest of the parser entries until matched again.

An example task using the native parser and the template above:

```yaml
- name: "Run command and parse with native"
  ansible.netcommon.cli_parse:
    command: show interface
      parser:
        name: ansible.netcommon.native
    set_fact: interfaces
```
would set the following `interfaces` fact for the device:

```yaml
Ethernet1/1:
    hardware: 100/1000/10000 Ethernet
    mac_address: 5254.005a.f8bd
    name: Ethernet1/1
    state:
    admin: up
    operating: up
Ethernet1/10:
    hardware: 100/1000/10000 Ethernet
    mac_address: 5254.005a.f8c6
<...>
```

About the task:
- Full module name is used for the task, including the collection (`ansible.netcommon.cli_parse`)
- The `command` key tell the module to run the command on the device or host, alternatively text from a previous command can be provided using the `text` key instead
- Information specific to the parser engine is provided in the `parser` key
- To use the `native` parser, the full name of the parsing engine, including it's collection, is provided as `name` (`ansible.netcommon.native`)
- The `cli_parse` module, by default, will look for the template in the templates directory as `{{ short_os }}_{{ command }}.yaml`. The `short_os` is derived from either the hosts `ansible_network_os` or `ansible_distribution`. The `command` spaces are replace with `_`.

#### Linux example

The native parser can also run commands and parse output from linux hosts.

Example command output of the `ip addr show` command

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether e8:6a:64:9d:84:19 brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 6e:c2:44:f7:41:e0 brd ff:ff:ff:ff:ff:ff permaddr d8:f2:ca:99:5c:82
```

Example native parser template stored as `templates/fedora_ip_addr_show.yaml`

```yaml
---
- example: '1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000'
  getval: |
    (?x)                                                # free-spacing
    \d+:\s                                              # the interface index
    (?P<name>\S+):\s                                    # the name
    <(?P<properties>\S+)>                               # the properties
    \smtu\s(?P<mtu>\d+)                                 # the mtu
    .*                                                  # gunk
    state\s(?P<state>\S+)                               # the state of the interface
  result:
    "{{ name }}":
        name: "{{ name }}"
        loopback: "{{ 'LOOPBACK' in stats.split(',') }}"
        up: "{{ 'UP' in properties.split(',')  }}"
        carrier: "{{ not 'NO-CARRIER' in properties.split(',') }}"
        broadcast: "{{ 'BROADCAST' in properties.split(',') }}"
        multicast: "{{ 'MULTICAST' in properties.split(',') }}"
        state: "{{ state|lower() }}"
        mtu: "{{ mtu }}"
  shared: True

- example: 'inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0'
  getval: |
   (?x)                                                 # free-spacing
   \s+inet\s(?P<inet>([0-9]{1,3}\.){3}[0-9]{1,3})       # the ip address
   /(?P<bits>\d{1,2})                                   # the mask bits
  result:
    "{{ name }}":
        ip_address: "{{ inet }}"
        mask_bits: "{{ bits }}"
```

With the following task:

```yaml
- name: Run command and parse
  ansible.netcommon.cli_parse:
    command: ip addr show
    parser:
      name: ansible.netcommon.native
    set_fact: interfaces
```

The follow fact would have been set as the `interfaces` fact for the host:

```yaml
lo:
  broadcast: false
  carrier: true
  ip_address: 127.0.0.1
  mask_bits: 8
  mtu: 65536
  multicast: false
  name: lo
  state: unknown
  up: true
enp64s0u1:
  broadcast: true
  carrier: true
  ip_address: 192.168.86.83
  mask_bits: 24
  mtu: 1500
  multicast: true
  name: enp64s0u1
  state: up
  up: true
<...>
```
About the task:
- Note the use of `shared` in the parser template, this allows the interface name to be used in subsequent parser entries
- Facts would have been gatherd prior to determine the `ansible_distribution` needed to locate the template. Alternatively, the `parse/template_path` could have been provided
- The use of examples and free-spacing mode with the regular expressions can make for a more-readable template

### Json parsing example

Although Ansible will natively convert serialized json to ansible native data when recognised, the `cli_parse` module can be used as well.

```yaml
- name: "Run command and parse as json"
  ansible.netcommon.cli_parse:
    command: show interface | json
    parser:
      name: ansible.netcommon.json
    register: interfaces
```

About the task:
- The `show interface | json` command would have been issued on the device
- The output would be set as the `interfaces` fact for the device

### NTC_templates

The ntc_templates python library includes pre-defined textfsm templates for parsing a variety of network device commands output.

Example task:
``` yaml
- name: "Run command and parse with ntc_templates"
  ansible.netcommon.cli_parse:
    command: show interface
    parser:
      name: ansible.netcommon.ntc_templates
    set_fact: interfaces
```

The follow fact would have been set as the `interface` fact for the host:

```yaml
interfaces:
- address: 5254.005a.f8b5
  admin_state: up
  bandwidth: 1000000 Kbit
  bia: 5254.005a.f8b5
  delay: 10 usec
  description: ''
  duplex: full-duplex
  encapsulation: ARPA
  hardware_type: Ethernet
  input_errors: ''
  input_packets: ''
  interface: mgmt0
  ip_address: 192.168.101.14/24
  last_link_flapped: ''
  link_status: up
  mode: ''
  mtu: '1500'
  output_errors: ''
  output_packets: ''
  speed: 1000 Mb/s
- address: 5254.005a.f8bd
  admin_state: up
  bandwidth: 1000000 Kbit
  bia: 5254.005a.f8bd
  delay: 10 usec
```

About the task:
- In this case the device's `ansible_network_os` was converted to the ntc_template format `cisco_nxos`.  Alternatively the `os` could have been provided with the `parser/os` key.
- The `cisco_nxos_show_interface.textfsm` template, included with the ntc_templates package, was used to parse the output
- Additional information about the `ntc_templates` python library can be foud here: https://github.com/networktocode/ntc-templates


### pyATS parsing engine

pyATS is part of the Cisco Test Automation & Validation Solution.  It includes many predefined parsers for a number of network platforms and commands.  The predefined parsers that are part of the pyATS package can be used with the `cli_parse` module.

Example task:
```yaml
- name: "Run command and parse with pyats"
  ansible.netcommon.cli_parse:
    command: show interface
    parser:
      name: ansible.netcommon.pyats
    set_fact: interfaces
```

The follow fact would have been set as the `interface` fact for the host:

```yaml
mgmt0:
  admin_state: up
  auto_mdix: 'off'
  auto_negotiate: true
  bandwidth: 1000000
  counters:
    in_broadcast_pkts: 3
    in_multicast_pkts: 1652395
    in_octets: 556155103
    in_pkts: 2236713
    in_unicast_pkts: 584259
    rate:
      in_rate: 320
      in_rate_pkts: 0
      load_interval: 1
      out_rate: 48
      out_rate_pkts: 0
    rx: true
    tx: true
  delay: 10
  duplex_mode: full
  enabled: true
  encapsulations:
    encapsulation: arpa
  ethertype: '0x0000'
  ipv4:
    192.168.101.14/24:
      ip: 192.168.101.14
      prefix_length: '24'
  link_state: up
  <...>
  ```

  About the task:
  - Because the `ansible_network_os` for the device was `cisco.nxos.nxos` it was provided to pyATS as `nxos`. The `cli_parse` modules converts the `ansible_network_os` automatically. Alternatively, the `parser/os` key can be used to set the OS.
  - Using a combination of the command and OS the pyATS used the following parser: https://pubhub.devnetcloud.com/media/genie-feature-browser/docs/#/parsers/show%2520interface
  - The `cli_parse` module sets `cisco.ios.ios` to `iosxe` for pyATS, This can be overidden with the `parser/os` key.
  - `cli_parse` uses only uses the predefined parsers in pyATS. The full documentation for pyATS can be found here: https://developer.cisco.com/docs/pyats/
  - The full list of pyATS included parsers can be found here: https://pubhub.devnetcloud.com/media/genie-feature-browser/docs/#/parsers




