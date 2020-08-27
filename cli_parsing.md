## Parsing semi-structured text with Ansible

As the scope of automation increases, it's not uncommon to encounter a device, host or platform that only supports a command-line interface and  commands issued returns semi-structured text.

With the 1.2.0 release of the ansible.netcommon collection, Ansible now has available a single module to both run commands and parse the output.

In addition to the native parsing engine, the cli_parse module works with several third party parsing engines.

## Why parse the text?

Some of the more common use cases for parsing semi-structured text into Ansible native data stuctures include:

- Used with the `when` clause to conditionally run other tasks or roles
- Configuration and operational state compliance checking used with the `assert` module
- Used with the `template` module to generate reports about configuration and operational state information
- Generate host, device or platform commands or configuration when used with templates and `command` or `config` modules
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

### Parsing with the native parsing engine

The native parsing engine is included with the `cli_parse` module. It uses data captured using regular expressions to populate the parsed data structure.

The native parsing engine requires a template be used to parse the command output. A native template is stored as a yaml file.


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
- The `ansible.netcommon.native` parsing engine is fully supported with a Red Hat Ansible Automation Platform subscription

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
    link/ether x2:6a:64:9d:84:19 brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether x6:c2:44:f7:41:e0 brd ff:ff:ff:ff:ff:ff permaddr d8:f2:ca:99:5c:82
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
- Facts would have been gatherd prior to determine the `ansible_distribution` needed to locate the template. Alternatively, the `parser/template_path` could have been provided
- The use of examples and free-spacing mode with the regular expressions can make for a more-readable template
- The `ansible.netcommon.native` parsing engine is fully supported with a Red Hat Ansible Automation Platform subscription


### Parsing json

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
- json support is provide primary for playbook consistancy
- The use of `ansible.netcommon.json` is fully supported with a Red Hat Ansible Automation Platform subscription

### Parsing with ntc_templates

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

The follow fact would have been set as the `interfaces` fact for the host:

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
- Red Hat Ansible Automation Platform subscription support is limited to the use of the ntc_templates public APIs as documented.


### Parsing with pyATS

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

The follow fact would have been set as the `interfaces` fact for the host:

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
  - Red Hat Ansible Automation Platform subscription support is limited to the use of the pyATS public APIs as documented.


### Parsing with textfsm

textfsm is a python module which implements a template based state machine for parsing semi-formatted text. Originally developed to allow programmatic access to information returned from the command line interface (CLI) of networking devices.


Example textfsm template stored as `templates/nxos_show_interface.textfsm`
```
Value Required INTERFACE (\S+)
Value LINK_STATUS (.+?)
Value ADMIN_STATE (.+?)
Value HARDWARE_TYPE (.*)
Value ADDRESS ([a-zA-Z0-9]+.[a-zA-Z0-9]+.[a-zA-Z0-9]+)
Value BIA ([a-zA-Z0-9]+.[a-zA-Z0-9]+.[a-zA-Z0-9]+)
Value DESCRIPTION (.*)
Value IP_ADDRESS (\d+\.\d+\.\d+\.\d+\/\d+)
Value MTU (\d+)
Value MODE (\S+)
Value DUPLEX (.+duplex?)
Value SPEED (.+?)
Value INPUT_PACKETS (\d+)
Value OUTPUT_PACKETS (\d+)
Value INPUT_ERRORS (\d+)
Value OUTPUT_ERRORS (\d+)
Value BANDWIDTH (\d+\s+\w+)
Value DELAY (\d+\s+\w+)
Value ENCAPSULATION (\w+)
Value LAST_LINK_FLAPPED (.+?)

Start
  ^\S+\s+is.+ -> Continue.Record
  ^${INTERFACE}\s+is\s+${LINK_STATUS},\sline\sprotocol\sis\s${ADMIN_STATE}$$
  ^${INTERFACE}\s+is\s+${LINK_STATUS}$$
  ^admin\s+state\s+is\s+${ADMIN_STATE},
  ^\s+Hardware(:|\s+is)\s+${HARDWARE_TYPE},\s+address(:|\s+is)\s+${ADDRESS}(.*bia\s+${BIA})*
  ^\s+Description:\s+${DESCRIPTION}
  ^\s+Internet\s+Address\s+is\s+${IP_ADDRESS}
  ^\s+Port\s+mode\s+is\s+${MODE}
  ^\s+${DUPLEX}, ${SPEED}(,|$$)
  ^\s+MTU\s+${MTU}.*BW\s+${BANDWIDTH}.*DLY\s+${DELAY}
  ^\s+Encapsulation\s+${ENCAPSULATION}
  ^\s+${INPUT_PACKETS}\s+input\s+packets\s+\d+\s+bytes\s*$$
  ^\s+${INPUT_ERRORS}\s+input\s+error\s+\d+\s+short\s+frame\s+\d+\s+overrun\s+\d+\s+underrun\s+\d+\s+ignored\s*$$
  ^\s+${OUTPUT_PACKETS}\s+output\s+packets\s+\d+\s+bytes\s*$$
  ^\s+${OUTPUT_ERRORS}\s+output\s+error\s+\d+\s+collision\s+\d+\s+deferred\s+\d+\s+late\s+collision\s*$$
  ^\s+Last\s+link\s+flapped\s+${LAST_LINK_FLAPPED}\s*$$
```

Example task:
```yaml
- name: "Run command and parse with textfsm"
  ansible.netcommon.cli_parse:
    command: show interface
    parser:
      name: ansible.netcommon.textfsm
    set_fact: interfaces
```

The follow fact would have been set as the `interfaces` fact for the host:

```yaml
- ADDRESS: X254.005a.f8b5
  ADMIN_STATE: up
  BANDWIDTH: 1000000 Kbit
  BIA: X254.005a.f8b5
  DELAY: 10 usec
  DESCRIPTION: ''
  DUPLEX: full-duplex
  ENCAPSULATION: ARPA
  HARDWARE_TYPE: Ethernet
  INPUT_ERRORS: ''
  INPUT_PACKETS: ''
  INTERFACE: mgmt0
  IP_ADDRESS: 192.168.101.14/24
  LAST_LINK_FLAPPED: ''
  LINK_STATUS: up
  MODE: ''
  MTU: '1500'
  OUTPUT_ERRORS: ''
  OUTPUT_PACKETS: ''
  SPEED: 1000 Mb/s
- ADDRESS: X254.005a.f8bd
  ADMIN_STATE: up
  BANDWIDTH: 1000000 Kbit
  BIA: X254.005a.f8bd
```

About the task:
- Because the `ansible_network_os` for the device was `cisco.nxos.nxos` it was converted to `nxos`. Alternatively the `parser/os` key can be used to manually provide the OS.
- The textfsm template name defaulted to `templates/nxos_show_interface.textfsm` using a combination of the OS and command run. Alternatively the `parser/template_path` key can be used to override the gernated template path.
- Detailed information about `textfsm` can be found here: https://github.com/google/textfsm
- textfsm was previously made available as a filter plugin. Ansible users should transition to `cli_parse`
- Red Hat Ansible Automation Platform subscription support is limited to the use of the textfsm public APIs as documented.


### Parsing with ttp

TTP is a Python library for semi-structured text parsing using templates.  TTP uses a jinja like syntax to limit the need for regular expressions.  User familiar with jinja templating may find TTP's template syntax familiar.

Example template stored as `templates/nxos_show_interfaces.ttp`

```jinja
{{ interface }} is {{ state }}
admin state is {{ admin_state }}{{ ignore(".*") }}
```
Example task:
```yaml
- name: "Run command and parse with ttp"
  ansible.netcommon.cli_parse:
    command: show interface
    parser:
      name: ansible.netcommon.ttp
    set_fact: interfaces
```

The follow fact would have been set as the `interfaces` fact for the host:

```yaml
- admin_state: up,
  interface: mgmt0
  state: up
- admin_state: up,
  interface: Ethernet1/1
  state: up
- admin_state: up,
  interface: Ethernet1/2
  state: up
```

About the task:
- The default template path `templates/nxos_show_interface.ttp` was generated using the `ansible_network_os` for the host and `command` provided
- TTP supports several additional variables that will be passed to the parser.  These include:
  - `parser/vars/ttp_init`: Additional parameter passed when the parser is initialized
  - `parser/vars/ttp_results`: Additional params used to influence the parser output
  - `parser/vars/ttp_vars`: Additional variables made available in the template
- Additioanl docuemtation about ttp can be found here: https://ttp.readthedocs.io


### Converting XML

Although Ansible contains a number of plugins that can convert XML to Ansible native data structures, `cli_parse` run command on devices that return XML and return the converted data in a single task.

Example task:
```yaml
- name: "Run command and parse as xml"
    ansible.netcommon.cli_parse:
      command: show interface | xml
      parser:
        name: ansible.netcommon.xml
  set_fact: interfaces
```
The follow fact would have been set as the `interfaces` fact for the host:

```yaml
nf:rpc-reply:
  '@xmlns': http://www.cisco.com/nxos:1.0:if_manager
  '@xmlns:nf': urn:ietf:params:xml:ns:netconf:base:1.0
  nf:data:
    show:
      interface:
        __XML__OPT_Cmd_show_interface_quick:
          __XML__OPT_Cmd_show_interface___readonly__:
            __readonly__:
              TABLE_interface:
                ROW_interface:
                - admin_state: up
                  encapsulation: ARPA
                  eth_autoneg: 'on'
                  eth_bia_addr: x254.005a.f8b5
                  eth_bw: '1000000'
```
About the task:

- Red Hat Ansible Automation Platform subscription support is limited to the use of the xmltodict public APIs as documented.


## Advanced Usage

The `cli_parse` module supports several features to support more complex uses cases.

### Provide a full template path

In the case the default template path needs to be overridden, it can be provided in the task

```yaml
- name: "Run command and parse with native"
  ansible.netcommon.cli_parse:
    command: show interface
    parser:
      name: ansible.netcommon.native
      template_path: /home/user/templates/filename.yaml
```

### Provide command to parser different than the command run

In the case the command run doesn't match the command the parser is expecting, it can be provided directly

```yaml
- name: "Run command and parse with native"
  ansible.netcommon.cli_parse:
    command: sho int
    parser:
      name: ansible.netcommon.native
      command: show interface
```

### Provide a custom OS value

Rather than using `ansible_network_os` or `ansible_distribution` for the generation of the template path or with the specified parser engine, it can be provided within the task.

```yaml
- name: Use ios instead of iosxe for pyats
  ansible.netcommon.cli_parse:
    command: show something
    parser:
      name: ansible.netcommon.pyats
      os: ios

- name: Use linux instead of fedora from ansible_distribution
  ansible.netcommon.cli_parse:
    command: ps -ef
    parser:
      name: ansible.netcommon.native
      os: linux
```

### Parse exisiting text

If the text needing to be parsed was collected earlier in the playbook or stored in a file, it can be provided to `cli_parse` as `text` instead of `command`.

```yaml
# using /home/user/templates/filename.yaml
- name: "Parse text from previous task"
  ansible.netcommon.cli_parse:
    text: "{{ output['stdout'] }}"
    parser:
      name: ansible.netcommon.native
      template_path: /home/user/templates/filename.yaml

 # using /home/user/templates/filename.yaml
- name: "Parse text from file"
  ansible.netcommon.cli_parse:
    text: "{{ lookup('file', 'path/to/file.txt') }}"
    parser:
      name: ansible.netcommon.native
      template_path: /home/user/templates/filename.yaml

# using templates/nxos_show_version.yaml
- name: "Parse text from previous task"
  ansible.netcommon.cli_parse:
    text: "{{ sho_version['stdout'] }}"
    parser:
      name: ansible.netcommon.native
      os: nxos
      command: show version
```

## Developer Notes

`cli_parse` can be used as an entry point for a cli_parser plugin in any collection.

Sample custom cli_parser plugin:

```python
from ansible_collections.ansible.netcommon.plugins.module_utils.cli_parser.cli_parserbase import (
    CliParserBase,
)

class CliParser(CliParserBase):
    """ Sample cli_parser plugin
    """

    # Use the follow extention when loading a template
    DEFAULT_TEMPLATE_EXTENSION = "txt"
    # Provide the contents of the template to the parse function
    PROVIDE_TEMPLATE_CONTENTS = True

    def myparser(text, template_contents):
      # parse the text using the template contents
      return {...}

    def parse(self, *_args, **kwargs):
        """ Std entry point for a cli_parse parse execution

        :return: Errors or parsed text as structured data
        :rtype: dict

        :example:

        The parse function of a parser should return a dict:
        {"errors": [a list of errors]}
        or
        {"parsed": obj}
        """
        template_contents = kwargs["template_contents"]
        text = self._task_args.get("text")
        try:
            parsed = myparser(text, template_contents)
        except Exception as exc:
            msg = "Custome parser returned an error while parsing. Error: {err}"
            return {"errors": [msg.format(err=to_native(exc))]}
        return {"parsed": parsed}
```

Example task:
```yaml
- name: Use a custom cli_parser
  ansible.netcommon.cli_parse:
    command: ls -l
    parser:
      name: my_organiztion.my_collection.custom_parser
```

About the plugin:
- Each cli_parser plugin requires a `CliParser` class
- Each cli_parser plugin requires a `parse` function
- Always return a dictionary with `errors` or `parsed`
- Place the custom cli_parser in plugins/cli_parsers directory of the collection
- See the current cli_parsers for additional examples: https://github.com/ansible-collections/ansible.netcommon/tree/main/plugins/cli_parsers










