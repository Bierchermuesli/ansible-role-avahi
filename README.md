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

## mDNS names in unicast DNS

Avahi is not a DNS server and dnsmasq cannot speak mDNS, so the usual
answers to "my Android phone / container / script cannot resolve
`foo.local`" are a second mDNS stack or a full RFC 8766 discovery proxy.
Neither is needed: the reflector host already holds every name it has seen.
With `avahi_hosts_file_enabled` a timer dumps that cache into a hosts(5)
file every minute, atomically and only when something changed:

```
100.88.88.41    Sonos-38420BF76138.local sonos-38420bf76138.mdns.example.net
```

hosts(5) is the one format every resolver reads, so the same file works
unchanged with most of them:

| Resolver | Hook it up | Picks up changes |
|---|---|---|
| Pi-hole 6 | drop the file into `/etc/pihole/hosts/` (the default path) | by itself |
| dnsmasq | `addn-hosts=/path/avahi.list` | `avahi_hosts_file_reload_command: systemctl reload dnsmasq` |
| CoreDNS | `hosts /path/avahi.list { reload 30s }` | by itself |
| Knot Resolver | `hints.add_hosts('/path/avahi.list')` | on reload |
| Blocky, AdGuard Home, systemd-resolved | hosts file sources | by itself |
| Unbound | no hosts(5) input; needs `local-data:` lines | not covered yet |

Entries disappear with the device. Only hosts whose records pass
`avahi_reflect_filters` are in the cache, so a device that advertises
nothing on the allowed list will not show up either.

The extra zone in `avahi_hosts_file_domains` is for clients that refuse to
send `.local` to DNS at all: systemd-resolved answers it locally with
NXDOMAIN, and Apple, Windows and most Linux desktops resolve `.local`
themselves via multicast and never ask. Android, containers and plain
`getaddrinfo()` on servers do ask, and get an answer.

This is name resolution only, not service discovery: a Chromecast app will
not find the Chromecast through DNS, but `curl http://chromecast.mdns.example.net:8008`
works from any segment.

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
| `avahi_hosts_file_enabled` | `false` | publish the mDNS cache as a hosts file |
| `avahi_hosts_file_path` | `/etc/pihole/hosts/avahi.list` | |
| `avahi_hosts_file_domains` | `[]` | extra zones to alias every host under |
| `avahi_hosts_file_ipv6` | `true` | include AAAA |
| `avahi_hosts_file_interval` | `1min` | |
| `avahi_hosts_file_reload_command` | `""` | run after a change, for servers that do not watch the file |

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
