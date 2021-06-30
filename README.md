## CLI RM Builder [Resource Module Builder for CLI-based modules]

### Overview

The `cli_rm_builder` is an Ansible Collection that helps developers scaffold and maintain Ansible Network Resource Modules.

**Capabilities**
- Use a pre-defined docstring (in YAML) to scaffold a resource module directory layout and initial class files in an Ansible Collection.
- Maintain the module `DOCUMENTATION` as the source of truth for the module argspec and use the builder to update the source files as needed.
- Generates working sample modules for both `<network_os>_<resource>` and `<network_os>_facts`


### Usage

```
pip install ansible-base
ansible-galaxy collection install git+https://github.com/ansible-network/cli_rm_builder.git
```

```yaml
run.yml
---
- hosts: localhost
  gather_facts: yes
  roles:
    - ansible_network.cli_rm_builder.run
```


#### Builing a new module/collection
```
ansible-playbook -e rm_dest=<destination for modules and module utils> \
                 -e collection_org=<collection_org> \
                 -e collection_name=<collection_name> \
                 -e docstring=</path/to/docstring> \
		 -e examples=</path/to/example>\
                 -e resource=<resource>
                 run.yml
```

#### Updating an existing module (regenerate argspec from docstring)
```
ansible-playbook -e rm_dest=<destination for modules and module utils> \
                 -e collection_org=<collection_org> \
                 -e collection_name=<collection_name> \
                 -e resource=<resource> \
                 run.yml
```

- `rm_dest`: The directory in which the files and directories for the resource module and facts modules should be placed
- `collection_org`: The organization of the collection
- `collection_name`: The name of the collection
- `resource`: The network resource targeted by the module
- `docstring`: The path to the file that contains docstring (in YAML)
- `examples`: The path to the file that contains examples (in txt)
- `network_os`: The value of network_os (defaults to `collection_name`)

### Docstrings

See the `docstrings` directory for an example.

***Note***: Within the scaffolded module file, the keys in the `RETURN` string have `type: dict` by default.
            These should be updated to `type: dict` if `config` key in module argspec is a list of dictionaries.

### Examples

**Collection directory layout**

- `network_os`: myos
- `resource`: interfaces

```
├── LICENSE
├── README.md
├── docs
├── meta
├── plugins
│   ├── action
│   │   └── __init__.py
│   ├── filter
│   │   └── __init__.py
│   ├── inventory
│   │   └── __init__.py
│   ├── module_utils
│   │   ├── __init__.py
│   │   └── network
│   │       ├── __init__.py
│   │       └── myos
│   │           ├── __init__.py
│   │           ├── argspec
│   │           │   ├── __init__.py
│   │           │   ├── facts
│   │           │   │   ├── __init__.py
│   │           │   │   └── facts.py
│   │           │   └── interfaces
│   │           │       ├── __init__.py
│   │           │       └── interfaces.py
│   │           ├── config
│   │           │   ├── __init__.py
│   │           │   └── interfaces
│   │           │       ├── __init__.py
│   │           │       └── interfaces.py
│   │           ├── facts
│   │           │   ├── __init__.py
│   │           │   ├── facts.py
│   │           │   └── interfaces
│   │           │       ├── __init__.py
│   │           │       └── interfaces.py
│   │           ├── rm_templates
│   │           │   ├── __init__.py
│   │           │   └── interfaces.py
│   │           └── utils
│   │               ├── __init__.py
│   │               └── utils.py
│   └── modules
│       ├── __init__.py
│       ├── myos_facts.py
│       └── myos_interfaces.py
└── tests
```

### Using collections in a playbook

Please refer to [using collections in a playbook](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html#using-collections-in-a-playbook) guide for detailed information.

### Resource Module Structure/Workflow

**Module**

`plugins/modules/<ansible_network_os>_<resource>.py`

- Import `module_utils` resource package and calls `execute_module` API
```
def main():
    result = <resource_package>(module).execute_module()
```

**Module Argspec**

`module_utils/network/<ansible_network_os>/argspec/<resource>/`

- Argspec for the resource. 
- The recommended way of updating the argspec is by updating the module docstring 
  first and then running the Resource Module Builder to update the argspec. This ensures
  that both the artifacts are always in sync.

**Facts**

`module_utils/network/<ansible_network_os>/facts/<resource>/`

- Populate facts for the resource.
- Entry in `module_utils/network/<ansible_network_os>/facts/facts.py` for `get_facts` API to keep
  `<ansible_network_os>_facts` module and facts gathered for the resource module in sync
  for every subset.
- Entry under the imports to register the Module's fact class- 
  `from ansible_collections.<ansible_network_org>.<ansible_network_os>.plugins.module_utils.network.
  <ansible_network_os>.facts.<resource>.<resource> import (<Resource>Facts,)`
- An entry in the global variable `FACT_RESOURCE_SUBSETS` is required in order to add it to the resource
  subsets as `<resource>=<Resource>Facts`.

**Parser Templates**

`module_utils/network/<ansible_network_os>/rm_templates/<resource>`

- Define a list of parser templates that the `NetworkTemplate` parent class in `ansible.netcommon` 
  uses to converts native configuration to Ansible structured data and vice versa.

- Example parser template:
```
    {  
        "name": "auto_cost",
        "getval": re.compile(
            r"""
            \s+auto-cost\sreference-bandwidth\s
            (?P<acrb>\d+)\s(?P<unit>\S+)$""",
            re.VERBOSE,
        ),
        "setval": (
            "auto-cost reference-bandwidth"
            " {{ auto_cost.reference_bandwidth }}"
            " {{ auto_cost.unit }}"
        ),
        "result": {
            "vrfs": {
                '{{ "vrf_" + vrf|d() }}': {
                    "auto_cost": {
                        "reference_bandwidth": "{{ acrb }}",
                        "unit": "{{ unit }}",
                    }
                }
            }
        },
    }
```

**Resource Config Package in module_utils**

`module_utils/network/<ansible_network_os>/<config>/<resource>/`

- Implement `execute_module` API that uses the existing device configuration (`have`), 
  the task input (`want`), and the `compare` functionality provided by the 
  parent class `ResourceModule` in `ansible.netcommon` to render and push 
  configuration commands to the target device.

- The task run result contains the following keys:
     - `before`: state of the `<resource>` in target device before module execution (as structured data)
     - `after`: state of the `<resource>` in target device after module execution (as structured data)
     - `commands`: list of commands sent to the target device
     - `changed`: `True` or `False` depending on whether the task run was idempotent
     - `gathered`: state of the `<resource>` in target device (as structured data) [only when `state`: `gathered`]
     - `parsed`: configuration passed through the `running_config` option as structured data [only when `state`: `parsed`]
     - `rendered`: provided configuration in the task as device native config lines [only when `state`: `rendered`]

- By default, the `empty_fact_val` is set to dict (`{}`). This needs to be updated to be list (`[]`) if `config` is a list
  of dictionaries.

    ***Note***: Please refer to [Network resource module states](https://docs.ansible.com/ansible/latest/network/user_guide/network_resource_modules.html#network-resource-module-states) for more information.

**Utils**

`module_utils/<ansible_network_os>/utils`.

- Utilities for the` <ansible_network_os>` platform.

### Developer Notes

The tests rely on a collection generated by the cli_rm_builder.
After changes to the builder, this test collection should be regenerated and the tests modified and run as needed.
To generate the collection after changes:

```
rm -rf tests/rmb_tests/collections/ansible_collections/myorg/myos
ansible-playbook -e docstring=docs/examples/docstrings/myos_interfaces.yaml \
                 -e rm_dest=tests/rmb_tests/collections/ansible_collections/myorg/myos \
                 -e resource=interfaces \
                 -e collection_org=myorg \
                 -e collection_name=myos \
                 run.yml
```
