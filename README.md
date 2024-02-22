# Netplan

[![Ansible Lint](https://github.com/MVladislav/ansible-netplan/actions/workflows/ansible-lint.yml/badge.svg)](https://github.com/MVladislav/ansible-netplan/actions/workflows/ansible-lint.yml)
[![Ansible Molecule Test](https://github.com/MVladislav/ansible-netplan/actions/workflows/ci.yml/badge.svg)](https://github.com/MVladislav/ansible-netplan/actions/workflows/ci.yml)

- [Netplan](#netplan)
  - [Role Variables](#role-variables)
  - [Dependencies](#dependencies)
  - [Example Playbook](#example-playbook)
  - [License](#license)

---

You can checkout [MVladislav - ansible-env-setup - playbooks](https://github.com/MVladislav/ansible-env-setup/tree/main/playbooks) for how i use it in general.

Tested with:

- Ubuntu 23.04

## Role Variables

```yml
netplan_install: no
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
          - "{{ ansible_default_ipv4.address }}/24"
        nameservers:
          addresses:
            - 1.1.1.1
            - 1.0.0.1
          search:
            - localdomain
        routes:
          - to: default
            via: "{{ ansible_default_ipv4.address.split('.')[0:3] | join('.') }}.1"
```

## Dependencies

Developed and testes with Ansible 2.14.4

## Example Playbook

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

MIT
