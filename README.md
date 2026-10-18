# avahi

[![CI](https://github.com/Bierchermuesli/ansible-role-avahi/actions/workflows/ci.yml/badge.svg)](https://github.com/Bierchermuesli/ansible-role-avahi/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Runs avahi-daemon as a pure mDNS reflector on a multi-homed host, so clients
in one VLAN discover AirPlay, Chromecast, HomeKit or printers in another.
Nothing is published about the host itself, nothing can be registered via
D-Bus, and `reflect-filters` limits what crosses segments.

The config is rendered before the package is installed: Debian's postinst
starts the daemon immediately, and with the stock config that would mean a
few seconds of avahi bound to every interface, WAN included.

## Usage

```yaml
- hosts: router
  become: true
  roles:
    - role: avahi
      vars:
        avahi_enabled: true
        avahi_allow_interfaces: [lan, iot]
        avahi_deny_interfaces: [wan, dmz]
        avahi_reflect_filters:
          - _googlecast._tcp.local
          - _airplay._tcp.local
          - _raop._tcp.local
          - _hap._tcp.local
          - _ipp._tcp.local
```

Open UDP/5353 on the allowed interfaces yourself. Exclude locally sourced
packets, otherwise the kernel's multicast loopback makes avahi's own
reflections show up as inbound traffic:

```
-A INPUT -i iot -p udp --dport 5353 -m addrtype ! --src-type LOCAL -j ACCEPT
```

Reflection only carries discovery. The actual connections (Chromecast
8008/8009, AirPlay 7000/7100 plus RTP, HomeKit high ports) still need
FORWARD rules between the segments.

## Variables

| Variable | Default | |
|---|---|---|
| `avahi_enabled` | `false` | role no-ops unless set |
| `avahi_allow_interfaces` | `[]` | interfaces avahi binds to; required when reflecting |
| `avahi_deny_interfaces` | `[]` | explicitly excluded, belt and braces |
| `avahi_reflector` | `true` | enable the reflector |
| `avahi_reflect_filters` | `[]` | service types to reflect, empty = everything |
| `avahi_reflect_ipv` | `false` | translate between IPv4 and IPv6 |
| `avahi_publish` | `false` | publish host records (hostname.local, addresses) |
| `avahi_use_ipv4`, `avahi_use_ipv6` | `true` | |
| `avahi_cache_entries_max` | `4096` | |
| `avahi_resolved_disable_mdns` | `true` | drop-in with `MulticastDNS=no` for systemd-resolved |

## Notes

- `reflect-filters` is a substring match on PTR/SRV/TXT records (avahi >= 0.8).
  A/AAAA host records always pass, so name resolution across segments keeps
  working. Without filters, `_ssh`, `_smb`, `_workstation` and the hostname of
  every device cross into the other segment too.
- Don't reflect into segments an attacker can reach. Reflected records leak
  hostnames and service types of every device in the allowed segments.
- `avahi-browse -art` on the host lists what the daemon currently knows,
  with the interface each record arrived on. A `.prom` collector for this is
  in [ansible-role-textfile-collectors].

[ansible-role-textfile-collectors]: https://github.com/Bierchermuesli/ansible-role-textfile-collectors
