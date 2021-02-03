# Private server to server network with ansible and wireguard

[![Ansible Role](https://img.shields.io/ansible/role/d/33136)](https://galaxy.ansible.com/mawalu/wireguard_private_networking)

This role allowes you to deploy a fast, secure and provider agnostic private network between multiple servers. This is usefull for providers that do not provide you with a private network or if you want to connect servers that are spread over multiple regions and providers.

## How

The role installs [wireguard](https://wireguard.com) on Debian or Ubuntu, creates a mesh between all servers by adding them all as peers and configures the wg-quick systemd service.

## Installation

Installation can be done using [ansible galaxy](https://galaxy.ansible.com/mawalu/wireguard_private_networking):

```
$ ansible-galaxy install mawalu.wireguard_private_networking
```

## Setup

Install this role, assign a `vpn_ip` variable to every host that should be part of the network and run the role. Plese make sure to allow the VPN port (default is 5888) in your firewall. Here is a small example configuration:

Optionally, you can set a `public_addr` on each host. This address will be used to connect to the wireguard peer instead of the address in the inventory. Useful if you are configuring over a different network than wireguard is using. e.g. ansible connects over a LAN to your peer.

If `expose_interface` is set, the subnet of the interface e.g. eth0 of the peer will be added to the AllowedIps section of this peer and the routing between Subnet and VPN will be set up. Be careful with conflicting subnets and remember to set a static route on your other devices in the subnet or on the default gateway.
`gateway_to` allows to specify a list of subnets that this peer can act as a gateway to. Wireguard routes packages to the peer with the most specific subnet settings. The rule prevents you from setting the subnet of a peer in the AllowedIps section of another peer which would absolutely lock you out. You should be very careful with conflicting subnets. If you can affort it, you sould give each peer that acts as a direct gateway to a subnet its own class C net. Even the cheapest consumer grade hardware is able to specify a custom class c net.

If a host does not have an endpoint that is reachable from the other hosts. E.g. behind a NAT that you cannot configure, behind a firewall or without a static address, you should set the flag `client_only`.
This will ensure the client opens a connection to every reachable host and tries to keep it alive. That way, you can access these peers and their networks.
Be aware, that if you have multiple hosts as `client_only` they cannot directly communicate to each other. You need at least one host that is reachable from both that can act as a relay. To achieve this, you can utilize the option `gateway_to`.


`expose_subnets` allows to specify a list of subnets that will be exposed by this peer. Along with the subnet of the `expose_interface` setting they will be added to the AllowedIps section of this peer.

```yaml
# inventory host file

wireguard:
  hosts:
    1.1.1.1:
      vpn_ip: 10.1.0.1/32
      public_addr: "example.com" # optional
      expose_interface: "eth0" # optional
    2.2.2.2:
      vpn_ip: 10.1.0.2/32
      client_only: true # optional
      expose_subnets: # optional
        - 192.168.3.0/24
        - 192.168.4.0/24
     3.3.3.3:
      vpn_ip: 10.1.0.3/32
      gateway_to: # optional
        - 10.1.0.0/24
        - 192.168.0.0/16
```

```yaml
# playbook

- name: Configure wireguard mesh
  hosts: wireguard
  remote_user: root
  roles:
    - mawalu.wireguard_private_networking
```

```yaml
# playbook (with client config)
- name: Configure wireguard mesh
  hosts: wireguard
  remote_user: root
  vars:
    client_vpn_ip: 10.1.0.100
    client_wireguard_path: "~/my-client-config.conf"
  roles:
    - mawalu.wireguard_private_networking
```

## Additional configuration

There are a small number of role variables that can be overwritten.

```yaml
wireguard_port: "5888" # the port to use for server to server connections
wireguard_path: "/etc/wireguard" # location of all wireguard configurations

wireguard_network_name: "private" # the name to use for the config file and wg-quick

wireguard_mtu: 1500 # Optionally a MTU to set in the wg-quick file. Not set by default. Can also be set per host

debian_enable_backports: true # if the debian backports repos should be added on debian machines

# Raspberry Pi Zero support
# Needs kernel headers and manual compilation of wireguard, opt in via flag, install `community.general` collection
# Caution: Might trigger a reboot.
allow_build_from_source: true

wireguard_sources_path: "/var/cache" # Location to clone the WireGuard sources if manual build is required

client_vpn_ip: "" # if set an additional wireguard config file will be generated at the specified path on localhost
client_wireguard_path: "~/wg.conf" # path on localhost to write client config, if client_vpn_ip is set

# a list of additional peers that will be added to each server
wireguard_additional_peers:
  - comment: martin
    ip: 10.2.3.4
    key: your_wireguard_public_key
  - comment: other_network
    ip: 10.32.0.0/16
    key: their_wireguard_public_key
    keepalive: 20
    endpoint: some.endpoint:2230

wireguard_post_up: "iptables ..." # PostUp hook command
wireguard_post_down: "iptables"   # PostDown hook command
```

## Testing

This role has a small test setup that is created using [molecule](https://github.com/ansible-community/molecule). To run the tests follow the molecule [install guide](https://molecule.readthedocs.io/en/latest/installation.html), ensure that a docker daemon runs on your machine and execute `molecule test`.

## Contributing

Feel free to open issues or MRs if you find problems or have ideas for improvements. I'm especially open for MRs that add support for additional operating systems and more tests.

