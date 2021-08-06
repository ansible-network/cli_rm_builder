# THE RESOURCE MODULE DOC

### Is your model ready?

```
git clone https://github.com/ansible-network/resource_module_models.git
```

Once the above repo is cloned go to the desired collection and add your model there. And talk to the team to get it _approved_.

##### Helpful Links-

- [Ansible 101: Part 1: In the beginning there was YAML](https://www.redhat.com/en/blog/ansible-101-part-1-beginning-there-was-yaml)
- [YAML syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)
- [Learn X in Y minutes](https://learnxinyminutes.com/docs/yaml/)

### Let’s get started with the Resource Module development.

At first, we would need a builder for our whole code base to get scaffolded from a tool - `cli_rm_builder`

```
pip install ansible-base

ansible-galaxy collection install git+https://github.com/ansible-network/cli_rm_builder.git
```

The collection _dir_ should have the following structure, follow the Github _namespaces_ for the setup under ansible collection

```
~/../../collections
❯ tree -L 3
├── collections
│   └── ansible_collections
│       ├── ansible
│       │   ├── netcommon
│       │   └── utils
│       ├── ansible_network
│       │   └── cli_rm_builder
│       ├── cisco
│       │   └── ios
│       ├── junipernetworks
│       │   └── junos
│       └── vyos
│           └── vyos

```

Once the cli_rm_builder and the target collection repo is cloned in the desired location we can work on the development of our module.
Using the tool to generate the base-level code with which we proceed.

```
❯ cat rm_builder_run.yml
---
- hosts: localhost
  gather_facts: yes
  roles:
    - ansible_network.cli_rm_builder.run
  vars:
    docstring:
    /../resource_module_models/models/{platform}/{{module}}/{platform}_{module_name}.yaml
    rm_dest:/home/{user}/{S}/{A}/collections/ansible_collections/{platform}/{platform}
    resource: {module_name}
    collection_org: {platform}
    collection_name: {platform}
    ansible_connection:local
```

```
❯ ansible-playbook rm_builder_run.yml
```

Post execution of the above command there should be few new files in your branch ready for the development of the resource module.

##### Important links at this point -

- Understanding RMs
  - [Ansible Network Resource Modules: Deep Dive on Return Values](https://www.ansible.com/blog/ansible-network-resource-modules-deep-dive-on-return-values)
  - [Developing network resource modules](https://docs.ansible.com/ansible/latest/network/dev_guide/developing_resource_modules_network.html)
  - [Getting started with Route Maps Resource Modules](https://www.ansible.com/blog/getting-started-with-route-maps-resource-modules)
    []()
- Debug Ansible resource module
  - [Debugging Ansible Network Modules with VSCode](https://docs.google.com/document/d/1KtgzLG8N4cyQ35NunaxaM_Gl37gFd0LObTtq6kRlLK4/edit)
  - [Debugging modules](https://docs.ansible.com/ansible/latest/dev_guide/debugging.html)
- Setup development environment
  - [Setting up Virtual Environments](https://docs.google.com/document/d/1eRUDIHZ6IYEegP9Z6HrhqWeGezM4n_8Pdxe8u-2TqN8/edit#heading=h.9qznvm65fex6)
  - [Creation Of Virtual environment & Ansible Installation](https://github.com/rohitthakur2590/ansible-dev-virtualenv)
    []()

# Introduction to RM files -

Looking at the collection where the new resource module is to be created after scaffolding the boiler plate code we should see new files already added -

`{platform}_{module_name}.py` - the entry point to the Resource Module code and the module documentation resides here. To change or update any argspec attribute during development we need to make the required change here directly in the docstring and then rerun the rm_builder_run.yml with \_docstring var commented out.

```
../collections/ansible_collections/{platform}/{platform}/plugins/modules/{platform}_{module_name}.py
```

`{module_name}.py` - under \_facts directory. The code in this file is responsible for converting native on-box configuration to Ansible structured data as per the argspec of the module by using the list of parsers defined in rm_templates/{module_name}.py.The on-box config can either be fetched from the target device or from the value set in the `running_config` key in the task when the module is run with `state: parsed`. The call to get data from the target device is often wrapped within a method. This is done to facilitate easier mocking when writing unit tests.

```
../collections/ansible_collections/{platform}/{platform}/plugins/module_utils/network/{platform}/facts/{module_name}/{module_name}.py
```

`facts.py` - You need to manually append the existence of your new resource module’s Facts class to the `FACTS_RESOURCE_SUBSET` dictionary in this file for facts to be generated as the call for facts is from a common instance i.e netcommon
The import -

```
from
ansible_collections.{platform}.{platform}.plugins.module_utils.network.{platform}.facts.{module_name}.{module_name}
import (
  Logging_globalFacts,
)
```

The entry under global Var -

```
FACT_RESOURCE_SUBSETS = dict({module_name}={module_name}Facts,)
```

```
../collections/ansible_collections/{platform}/{platform}/plugins/module_utils/network/{platform}/facts/facts.p
y
```

`logging_global.py` - The \_argspec file is the python level representation of your model. You may never need to edit it manually, change the model in the module file and cli_rm_builder should take care of updating it.

```
../collections/ansible_collections/{platform}/{platform}/plugins/module_utils/network/{platform}/argspec/{module_name}/{module_name}.py
```

`logging_global.py` - the \_rm_template\* is one of the most vital components of a Resource Module since the conversion of native on-box configuration to structured data and vice-versa is facilitated by the parser templates that are defined in this file. Time to spin up regex/ jinja templating skills for this file. The better they are the easier it would be to get a higher score in the module's Unit Test Coverage (UTC).

```
../collections/ansible_collections/{platform}/{platform}/plugins/module_utils/network/{platform}/rm_templates/{module_name}.py
```

##### Helpful Links

- Regex parsers
  - [Pythex Regex parser](https://pythex.org/)
  - [Regex101](https://regex101.com/)
- Jinja2 parsers
  - [J2 Live Parser](http://jinja.quantprogramming.com/)
  - [Ansible Template Tester](https://ansible.sivel.net/test/)

`Logging_global.py` - the \_config file contains all the core logic of how the execution should behave in various states. In here you get _want_ [the playbook] and _have_ [the config that came from the facts rendered]
All the states and their comparison logic goes here.

```
/home/sagpaul/Work/bannerNconfig/collections/ansible_collections/{platform}/{platform}/plugins/module_utils/network/{platform}/argspec/{module_name}/{module_name}.py
```

And, You might see a couple of *\_*init*\_*.py files generated,required for Ansible tests to pass!

## PHASE - 1 Gathering facts from the target device

The first step in building a resource module is to write facts code that converts device native configuration to structured data. This is done by comparing the device config with a set of pre-defined “Parser Templates” that define regexes to parse the native config.

Both the list of templates and the config are fed to an object of the NetworkTemplate class, on which the `parse()` method is then invoked. The output of the `parse()` method is semi-structured data that might need some additional updates to match the module’s argspec format.

Before finally rendering this data as facts, it is validated against the module’s argspec by the validate_config() method which fails if the data does not match the schema defined in the argspec.

## Anatomy of a Parser Template:

`name`
The name or unique identifier of the parser template.

`getval`
A regular expression using named capture groups to store the extracted data.Here
goes a regex that can break a command or a part of the command that is read from the device config and this contributes to generating facts from device native config. NOTE- This regex is not for validation, as this will not be used while forming the commands so we can just use simple regex forms to fragment the command and assign it to variables to use while we make our facts.

`setval`
This is used to generate device native config from Ansible structured data. It can either be a Python function or a Jinja2 template.

`result`-
A data tree, populated as a template, from the parsed data.This is where we generate the facts with the help of variables that are formed via the regex fragmentation we did in _getval_ It may match the part of facts the whole parser is written for i.e argspec i.e model.

`remval`-
(Optional) This is used to specify a command to negate an attribute. It can either be a Python function or a Jinja2 template. This is usually not needed, as in most cases simply negating the command generated by `setval` does the job.
However, this comes in handy when the command to remove an attribute is significantly different from `setval`.

`compval`-
(Optional) This is to be used for a complex model where parsers are broken down into multiple ones and are then referenced like namespaces. By default, the name of the parser itself is used to extract an attribute for want and have dictionaries.

For example:

```
want = {'k1': {'k2': {'k3': 'newval', ‘k4’: 'anotherval'}}}
have = {'k1': {'k2': {'k3': 'oldval', ‘k4’: 'anotherval'}}}
```

With parser name `'k1.k2.k3'`, the RMEngineBase will extract the value of the nested key `'k3'` from both _want_ and _have_ and then compare it in order to decide if an update is required. In this case, it will compare `'k3': 'newval'` and `'k3': 'oldval'`. However, if the parser template has `compval: k1.k2` defined, the value of the key `k2`(which is a dictionary itself) will be used. So here, the comparison will happen between `'k2': {'k3':'newval', k4: 'anotherval'}` and `{'k3': 'oldval', k4: 'anotherval'}`

`shared`-
(Optional) The shared key makes the parsed values available to the rest of the parser
entries until matched again.This enables the data/result of the parser to be shared among other parsers for reuse.

Example parsers -

```
vyos@vyos:~$ show configuration commands | grep syslog
set system syslog console facility all
set system syslog console facility local7 level 'err'
set system syslog console facility news level 'debug'
```

Given the set of commands the parser _can_ look like -
ref : [Vyos logging global model](https://github.com/ansible-network/resource_module_models/blob/master/models/vyos/logging_global/vyos_logging_global.yaml)

```
PARSERS =[{
            "name": "console.facilities",
            "getval": re.compile(
                r"""
                ^set\ssystem\ssyslog\sconsole\sfacility
                (\s(?P<facility>all|auth|authpriv|cron|daemon|kern|lpr|mail|mark|news|protocols|security|syslog|user|uucp|local[0-7]))?
                (\slevel\s(?P<level>'(emerg|alert|crit|err|warning|notice|info|debug|all)'))?
                $""", re.VERBOSE),
            "setval": tmplt_params,
            "remval": "system syslog console facility {{ console.facilities.facility }}",
            "result": {
                "console": {
                    "facilities": [{
                        "facility": "{{ facility }}",
                        "severity": "{{ level }}",
                    }, ]
                }
            }
        },]
```

Having setval ready at this point is not required, we can start off by executing our first playbook

```
---
- name: check GATHERED state
  hosts: your.host.name
  gather_facts: no
  tasks:
  - name: Gather logging config
    vyos.vyos.vyos_logging_global:
    state: gathered
```

```
---
- name: check PARSED state
  hosts: your.host.name
  gather_facts: no
  tasks:
  - name: Parse the provided configuration
    register: result
    vyos.vyos.vyos_logging_global:
      running_config: "{{ lookup('file', 'raw_vyos.cfg') }}"
      state: parsed
```

`raw_vyos.cfg`

This is just a flat file that holds the config that we get after the show commands are executed on the target device.

You should have a working facts code as of now! And the _gathered & parsed_ state should work before you proceed further.

##### Note

If there is a _list of items_ in the generated facts, it is suggested to sort them before they are rendered, in order to to get consistent output across different Python versions. This also helps with assertions while working on Unit or IntegrationTests.

## PHASE - 2 THE CONFIG / MERGED and other STATEs

Let’s talk about the different [states](https://docs.ansible.com/ansible/latest/network/user_guide/network_resource_modules.html#network-resource-module-states)!

`WANT` is basically the exact representation of the task, which is accessible in the config code.

`HAVE` is the on-box config as structured data that is rendered by the facts code output is just a representation of the commands that are formed based on want/have and states

`MERGED` -

```
|     WANT     |    HAVE     |    Output    |  Comment  |
| :----------: | :---------: | :----------: | :-------: |
| {A, B, C, D} |   {A,B,E}   |    {C,D}     |  Changed  |
|      {}      | {A,B,C,D,E} |      {}      | No change |
| {A, B, C, D} |     {}      | {A, B, C, D} |  Changed  |
```

`REPLACED`- _rA <- replace A_

```

|     WANT     |     HAVE     |     Output     |   Comment   |
| :----------: | :----------: | :------------: | :---------: |
| {A, B, C, D} | {A, B, E, F} | {rA, rB, E, F} | Changed A,B |

```

`OVERRIDDEN`- _nA - Negate A_

```
|     WANT     |    HAVE     |        Output         |  Comment  |
| :----------: | :---------: | :-------------------: | :-------: |
| {A, B, C, D} |   {A,B,E}   |     {A, B, nE, D}     |  Changed  |
|      {}      | {A,B,C,D,E} |          {}           | No change |
| {A, B, C, D} |     {}      |     {A, B, C, D}      |  Changed  |
| {A, B, C, D} |   {E,F,G}   | {nE,nF,nG,A, B, C, D} |  Changed  |
```

`DELETED`-

```
|     WANT     |    HAVE     |      Output      |  Comment  |
| :----------: | :---------: | :--------------: | :-------: |
| {A, B, C, D} |   {A,B,E}   |     {nA, nB}     |  Changed  |
|      {}      | {A,B,C,D,E} | {nA,nB,nC,nD,nE} | No change |
| {A, B, C, D} |     {}      |        {}        | No Change |
| {A, B, C, D} |   {E,F,G}   |        {}        | No Change |
```

`RENDERED`- Pass in a config with the rendered state it is supposed to tell you all the set of commands that would be formed on the supplied config (without actually connecting to the target device), it is different from check mode.

`PARSED`- The parsed state is just opposite to the rendered state it tells you how the invocation/facts would look like when you supply the running_config/ raw config from a device.

##### Some important links at this point-

- Parsing semi-structured text with Ansible
  - [Parsing the CLI](https://docs.ansible.com/ansible/latest/network/user_guide/cli_parsing.html)
- Working with command output and prompts in network modules
  - [Handling prompts](https://docs.ansible.com/ansible/latest/network/user_guide/network_working_with_command_output.html)
- Network Debug and Troubleshooting Guide
  - [Debugging network Modules](https://docs.ansible.com/ansible/latest/network/user_guide/network_debug_troubleshooting.html)

## Conclusion

On the config side code, you get a lot of creative liberty to handle the _want_ and _have,_ compare them and make them work as per the states. The final set of commands that need to be executed on the target nodes are formed here, based on the _setval_ values defined within the parsers.
Implementation of _list to dict_ on every attribute is imp on the entry point of config code before it starts getting processed on the basis of states. As the \_compare() method understands it better.
A comparison of two dictionary of dictionaries is easier and more efficient than a comparison of two lists of dictionaries. Hence, to optimally leverage the RMEngineBase, it is important that we convert all lists to dicts to dicts of dicts before starting with the comparison process.

##### Example list to dict :

- nxos.route_maps [\_route_maps_list_to_dict](https://github.com/ansible-collections/cisco.nxos/blob/main/plugins/module_utils/network/)
- eos.bpg_global [\_bgp_global_list_to_dict](https://github.com/ansible-collections/arista.eos/blob/main/plugins/module_utils/network/eos/config/bgp_global/bgp_global.py#L365-L396)
- iosxr.bgp_global [\_bgp_list_to_dict](https://github.com/ansible-collections/cisco.iosxr/blob/main/plugins/module_utils/network/iosxr/config/bgp_global/bgp_global.py#L376-L407)

With the config development in place, there are few things to keep a note of to make the code clean and reusable by the rest of the modules within the same platform,

```
..collections/ansible_collections/{platform}/{platform}/plugins/module_
utils/network/{platform}/utils/utils.py
```

At the above path, `utils.py` creates a set of defined methods that includes flattening the config or processing the list_to_dict operations for that platform.Adding a generic method here and making the whole module reuse that existing code adds up to the code quality.

Back to config, how the _setvals_ are picked up after the compare method is at a point to generate the commands that are finally used to apply the necessary comparison.

So, the compare method on comparison of two dicts refers to it by the name of the parsers and tries to match that with the defined list of parsers in the config code. Adding the parser names in namespace format helps the compare method to reduce the namespace based on the dictionary it is looking at and does the setval computation on the basis of that. There is no direct relation between the _result_ key and _setval_ key int he parsers. The data available at setvals to generate the command may or may not be aligned with the facts/results from parsers. It depends on the flattening logic written in config which molds our have and want data or better I say wantd and haved to be easily compared. Here if a dict contains

```
haved = { 'key1' : { 'key2' : 100 , }}
wanted = { 'key1' : { 'key2' : 10 , }}
```

A parser named Something will do the compare and push the whole dict for being processed in setvals. Whereas a parser named Something.Anything will do the comparison on a level under Something as per the above example.

Happy contribution!
