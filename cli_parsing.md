## Parsing semi-structured text with Ansible

As the scope of automation increases, it's not uncommon to encounter a device or platform that only supports a command-line interface and  commands issued returns semi-structured text.

With the 1.2.0 release of the ansible.netcommon collection, ansible now has available a single module to both run commands and parse the output of these devices.

In addition to the native parsing engine, the cli_parse module works with several third party parsing engines.

## Why parse the text?

Some of the more common use cases for parsing semi-structured text into ansible native data stuctures include

- Used with the `when` clause to conditionally run other tasks or roles
- Configuration and operational state compliance checking used with the `assert` module
- Used with the `template` module to generate reports about configuration and operational state information
- Generate device commands or configuration when used with templates and `command` of `config` modules
- Suppliment native facts information when used along side current platform `facts` modules
- Populate the ansible inventory or a CMDB with new structured data

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

Using the output of the `show interface` from a network device, each of the parsers engines are documented below

example output:
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

### native

The native parsing engine is included with the cli_parse module. It uses data captured using regular expressions to populate the parsed data structure.

The native parsing engine requires a template be used to parse the command output. A native template stored as a yaml file.

An example template:

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

- example: An example line of the text line to be parsed
- getval: A regular expression using named capture groups to store the extracted data
- result: A data tree, populated as a template, from the parsed data
- shared: (optional) The shared key makes the parsed values available to the rest of the parser entries until matched again.

An example task using the native parser and the template above:

```yaml
- name: "Run command and parse with native"
  ansible.netcommon.cli_parse:
    command: show interface
      parser:
        name: ansible.netcommon.native
    set_fact: interfaces
```
would produce the following data:

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
- The `cli_parse` module, by default, will look for the template in the templates directory as `{{ short_os }}_{{ command }}.yaml`. The `short_os` is derived from either the hosts `ansible_network_os` or `ansible_distribution`. The `command` space are replace with `_`.
- The example template above was stored as `templates/nxos_show_interface.yaml`
