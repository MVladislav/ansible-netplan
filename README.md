# Role Name

A brief description of the role goes here.

## Requirements

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

## Role Variables

```yml
netplan_remove_existing: no
netplan_service_name: default
# for template
netplan_interface_name: eth0
netplan_dhcp_4: yes
netplan_dhcp_6: no
netplan_addresses:
  - "{{ ansible_host }}/24"
netplan_gateway_4: "{{ ansible_host.split('.')[0:3] | join('.') }}.1"
netplan_router: null
# netplan_router:
#   - to: 0.0.0.0/0
#     via: "{{ ansible_host.split('.')[0:3] | join('.') }}.1"
#     metric: 100
netplan_nameserver_addresses:
  - 1.1.1.1
  - 1.0.0.1
```

## Dependencies

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

## Example Playbook

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yml
- hosts: clients
  roles:
    - role: netplan
      netplan_service_name: "{{ service_name }}"
      netplan_setup: yes
      netplan_remove_existing: yes
      netplan_interface_name: eth0
      netplan_dhcp_4: yes
      netplan_dhcp_6: no
      netplan_addresses:
        - "{{ ansible_host }}/24"
      netplan_gateway_4: "{{ ansible_host.split('.')[0:3] | join('.') }}.1"
      netplan_nameserver_addresses:
        - 1.1.1.1
        - 1.0.0.1
```

## License

GNU AFFERO GENERAL PUBLIC LICENSE

## Author Information

MVladislav
