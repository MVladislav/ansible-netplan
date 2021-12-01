# Role Name

A brief description of the role goes here.

## Requirements

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

## Role Variables

```yml
netplan_remove_existing: no
netplan_service_name: default

# NetworkManager | networkd
netplan_renderer: networkd
netplan_version: 2

netplan_dir: /etc/netplan
netplan_file: "00-{{ netplan_service_name }}.yaml"

# for template
netplan_configuration:
  network:
    ethernets:
      eth0:
        dhcp4: yes
        dhcp6: no
        addresses:
          - "{{ ansible_host }}/24"
        nameservers:
          addresses:
            - 1.1.1.1
            - 1.0.0.1
          search:
            - localdomain
        routes:
          - to: default
            via: "{{ ansible_host.split('.')[0:3] | join('.') }}.1"
```

## Dependencies

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.

## Example Playbook

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yml
- hosts: clients
  roles:
    - role: netplan
      netplan_remove_existing: no
      netplan_service_name: default
      netplan_renderer: networkd
      netplan_version: 2
      netplan_dir: /etc/netplan
      netplan_file: "00-{{ netplan_service_name }}.yaml"
      netplan_configuration:
        network:
          ethernets:
            eth0:
              dhcp4: yes
              dhcp6: no
              addresses:
                - "{{ ansible_host }}/24"
              nameservers:
                addresses:
                  - 1.1.1.1
                  - 1.0.0.1
              gateway4: "{{ ansible_host.split('.')[0:3] | join('.') }}.1"
```

## License

GNU AFFERO GENERAL PUBLIC LICENSE

## Author Information

MVladislav
