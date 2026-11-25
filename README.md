# Netplan

[![Ansible Lint](https://github.com/MVladislav/ansible-netplan/actions/workflows/ansible-lint.yml/badge.svg)](https://github.com/MVladislav/ansible-netplan/actions/workflows/ansible-lint.yml)
[![Ansible Molecule Test](https://github.com/MVladislav/ansible-netplan/actions/workflows/ci.yml/badge.svg)](https://github.com/MVladislav/ansible-netplan/actions/workflows/ci.yml)

- [Netplan](#netplan)
  - [Role Variables](#role-variables)
  - [Dependencies](#dependencies)
  - [Example Playbook](#example-playbook)
  - [License](#license)

---

Ansible role to install and configure [Netplan](https://netplan.io/).

You can check out [MVladislav - ansible-env-setup - playbooks](https://github.com/MVladislav/ansible-env-setup/tree/main/playbooks) for examples of how I use it.

Tested with:

- Ubuntu 22.04
- Ubuntu 24.04

## Role Variables

Default values (see `defaults/main.yml`):

```yaml
netplan_install: false
netplan_remove_existing: false
netplan_service_name: default

# NetworkManager | networkd
netplan_renderer: networkd
netplan_version: 2

netplan_dir: /etc/netplan
netplan_file: "00-{{ netplan_service_name }}.yaml"

# Default interface used by the basic configuration below.
netplan_default_iface: >-
  {{ (ansible_facts | default({})).get('default_ipv4', {}).get('interface')
     or (ansible_facts | default({})).get('default_ipv6', {}).get('interface')
     or 'eth0' }}

# Basic default: configure the default interface with DHCP for IPv4 and IPv6.
netplan_configuration:
  network:
    ethernets: "{{ {netplan_default_iface: {'dhcp4': true, 'dhcp6': true}} }}"
```

Notes:

- `netplan_version` and `netplan_renderer` are rendered into the generated Netplan file by the role template.
- `netplan_configuration.network` can contain sections supported by the template: `ethernets`, `wifis`, `bonds`, `bridges`, and `vlans`.
- The default configuration is intended as a minimal, safe starting point (DHCPv4/DHCPv6 on the detected default interface). Override `netplan_configuration` for static or advanced setups.

## Dependencies

- Ansible >= 2.14 (see `meta/main.yml`)

## Examples

> Most examples below rely on Ansible facts from the target host. Run with `gather_facts: true` (default) so values like interface name and MAC address can be detected automatically.

### 1) Basic default (DHCPv4 + DHCPv6 on the detected default interface)

This is the role default (see `defaults/main.yml`): detect the default interface and enable DHCP for IPv4 and IPv6.

```yaml
- hosts: all
  become: true
  roles:
    - role: netplan
      vars:
        netplan_install: true
        netplan_remove_existing: false
```

### 2) Normal server example (DHCP on the network, networkd)

For a typical server on a network with DHCP, detect the default interface and MAC at runtime, match the interface by MAC, and set a stable interface name.

```yaml
- hosts: servers
  become: true
  roles:
    - role: netplan
      vars:
        pl_a_netplan_setup: true # optional, only if your playbook uses this flag
        netplan_remove_existing: false
        netplan_renderer: networkd
        netplan_default_iface: >-
          {{ (ansible_facts | default({})).get('default_ipv4', {}).get('interface')
             or (ansible_facts | default({})).get('default_ipv6', {}).get('interface')
             or 'eth0' }}
        netplan_default_mac: >-
          {{ (ansible_facts | default({})).get('default_ipv4', {}).get('macaddress')
             or (ansible_facts | default({})).get('default_ipv6', {}).get('macaddress')
             or '' }}
        netplan_iface_cfg_server:
          dhcp4: true
          dhcp6: true
          match:
            macaddress: "{{ netplan_default_mac }}"
          set-name: "{{ netplan_default_iface }}"
          # Optional: add static nameservers/addresses/routes if required by your network.
          # nameservers:
          #   addresses:
          #     - 1.1.1.1
          #     - 1.0.0.1
        netplan_configuration:
          network:
            ethernets: "{{ {netplan_default_iface: netplan_iface_cfg_server} }}"
```

### 3) Hetzner-style server example (IPv6 routed via link-local gateway, networkd)

Some providers (e.g. Hetzner dedicated/root servers) require a provider-specific IPv6 setup, commonly a static IPv6 address/prefix and a default IPv6 route via the link-local gateway `fe80::1` with `on-link: true`. Adjust addresses, DHCP settings, and nameservers to match your assigned network configuration.

```yaml
- hosts: servers
  become: true
  roles:
    - role: netplan
      vars:
        pl_a_netplan_setup: true # optional, only if your playbook uses this flag
        netplan_remove_existing: false
        netplan_renderer: networkd
        netplan_default_iface: >-
          {{ (ansible_facts | default({})).get('default_ipv4', {}).get('interface')
             or (ansible_facts | default({})).get('default_ipv6', {}).get('interface')
             or 'eth0' }}
        netplan_default_mac: >-
          {{ (ansible_facts | default({})).get('default_ipv4', {}).get('macaddress')
             or (ansible_facts | default({})).get('default_ipv6', {}).get('macaddress')
             or '' }}
        netplan_iface_cfg_server:
          dhcp4: true
          dhcp6: false
          match:
            macaddress: "{{ netplan_default_mac }}"
          set-name: "{{ netplan_default_iface }}"
          addresses:
            - "<YOUR_IPV6_ADDRESS_OR_PREFIX>/64" # replace with the address/prefix assigned by your provider
          routes:
            - on-link: true
              to: default
              via: "fe80::1"
          nameservers:
            addresses:
              - 2a01:4ff:ff00::add:1
              - 2a01:4ff:ff00::add:2
              - 185.12.64.1
              - 185.12.64.2
        netplan_configuration:
          network:
            ethernets: "{{ {netplan_default_iface: netplan_iface_cfg_server} }}"
```

Notes:

- This is **provider/network-specific**; verify the exact IPv6 address/prefix, gateway behavior, and DNS servers for your server.
- Prefer `routes` with `to: default` for default gateways (instead of legacy `gateway4`/`gateway6`).
- `match` + `set-name` can be disruptive on live systems; validate carefully (console/out-of-band access recommended when changing remote networking).

### 4) Client example (NetworkManager, deterministic UUID, custom netplan filename)

A NetworkManager-based client example with a deterministic (stable/idempotent) UUID, interface match by MAC, and a custom `90-...` netplan filename for predictable precedence.

```yaml
- hosts: clients
  become: true
  roles:
    - role: netplan
      vars:
        pl_a_netplan_setup: true # optional, only if your playbook uses this flag
        netplan_remove_existing: false
        netplan_renderer: NetworkManager
        netplan_default_iface: >-
          {{ (ansible_facts | default({})).get('default_ipv4', {}).get('interface')
             or (ansible_facts | default({})).get('default_ipv6', {}).get('interface')
             or 'eth0' }}
        netplan_default_mac: >-
          {{ (ansible_facts | default({})).get('default_ipv4', {}).get('macaddress')
             or (ansible_facts | default({})).get('default_ipv6', {}).get('macaddress')
             or '' }}
        # Deterministic UUID per host/interface (stable across runs). Use a fixed UUID if you prefer.
        netplan_nm_uuid: "{{ (inventory_hostname ~ '-' ~ netplan_default_iface) | to_uuid }}"
        netplan_service_name: "nm-{{ netplan_default_iface }}"
        netplan_file: "90-{{ netplan_service_name }}.yaml"
        netplan_iface_cfg_client:
          renderer: NetworkManager
          dhcp4: true
          dhcp6: true
          match:
            macaddress: "{{ netplan_default_mac }}"
          set-name: "{{ netplan_default_iface }}"
          networkmanager:
            uuid: "{{ netplan_nm_uuid }}"
            name: "{{ netplan_default_iface }}"
            # Optional NetworkManager passthrough settings:
            # passthrough:
            #   ipv6.ip6-privacy: "-1"
        netplan_configuration:
          network:
            ethernets: "{{ {netplan_default_iface: netplan_iface_cfg_client} }}"
```

Notes:

- Use a unique/stable UUID per connection profile. `{{ (inventory_hostname ~ '-' ~ netplan_default_iface) | to_uuid }}` is a good inventory-safe default; you can also set a literal UUID generated once with `uuidgen`.
- When `netplan_renderer != 'networkd'`, this role stops/disables `systemd-resolved` (see `tasks/main.yml`). Ensure this matches your host/network design.
- Set `netplan_remove_existing: true` only if you intentionally want to remove other netplan YAML files in `{{ netplan_dir }}` (except the generated file).

### 5) Dynamic address/prefix and default route from target facts (routes-based)

If you want to pin the target's currently detected IP configuration (instead of DHCP), use Ansible facts to fill address/prefix and default route at runtime. Prefer `routes` with `to: default`.

#### IPv4 example

```yaml
- hosts: all
  become: true
  roles:
    - role: netplan
      vars:
        netplan_default_iface: >-
          {{ (ansible_facts | default({})).get('default_ipv4', {}).get('interface')
             or (ansible_facts | default({})).get('default_ipv6', {}).get('interface')
             or 'eth0' }}
        netplan_configuration:
          network:
            ethernets: >-
              {{ {
                netplan_default_iface: {
                  'dhcp4': false,
                  'dhcp6': false,
                  'addresses': [
                    (ansible_facts | default({})).get('default_ipv4', {}).get('address')
                    ~ '/' ~ ((ansible_facts | default({})).get('default_ipv4', {}).get('prefix', 24) | string)
                  ],
                  'routes': [
                    {
                      'to': 'default',
                      'via': (ansible_facts | default({})).get('default_ipv4', {}).get('gateway')
                    }
                  ],
                  'nameservers': {
                    'addresses': ['1.1.1.1', '1.0.0.1']
                  }
                }
              } }}
```

#### IPv6 example

```yaml
- hosts: all
  become: true
  roles:
    - role: netplan
      vars:
        netplan_default_iface: >-
          {{ (ansible_facts | default({})).get('default_ipv6', {}).get('interface')
             or (ansible_facts | default({})).get('default_ipv4', {}).get('interface')
             or 'eth0' }}
        netplan_configuration:
          network:
            ethernets: >-
              {{ {
                netplan_default_iface: {
                  'dhcp4': false,
                  'dhcp6': false,
                  'addresses': [
                    (ansible_facts | default({})).get('default_ipv6', {}).get('address')
                    ~ '/' ~ ((ansible_facts | default({})).get('default_ipv6', {}).get('prefix', 64) | string)
                  ],
                  'routes': [
                    {
                      'to': 'default',
                      'via': (ansible_facts | default({})).get('default_ipv6', {}).get('gateway')
                    }
                  ],
                  'nameservers': {
                    'addresses': ['2606:4700:4700::1111', '2606:4700:4700::1001']
                  }
                }
              } }}
```

These patterns are useful as a mostly-static baseline: keep structure explicit and auto-fill host-specific interface/address/gateway values at Ansible runtime.

## License

MIT
