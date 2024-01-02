# genwg
genwg creates wireguard client and server configuration files. it handles:
- creating local zones for the servers with A and PTR records for the client to
  be used with BIND9. this rids the need to know the ip address for the client
  to be noted down and allows for listing all of the clients for a server via
  doing a zone transfer `dig axfr .local_zone @<ip.of.server.iface>`
  __(all client platforms)__
- configuration creation for wireguard over faketcp w/ udp2raw for both servers
  and clients. __(linux and android only)__
- allows clients to incorporate the server's recursive DNS resolver into their
  local BIND instance so that they get to keep their local zones while
  forwarding the root zone requests to the wireguard server and preventing
  leaks. __(linux only)__

## installation
```sh
git clone --depth=1 https://github.com/gottaeat/genwg
cd genwg/
pip install .
```

## usage
```sh
genwg --help
```

## yaml specification
### servers
| key | necessity | description |
|---|---|---|
| name | required | (str) name for the interface |
| proto | required | (str) can be udp or tcp |
| hostname | required if --bind is specified | (str) hostname of the machine (necessary for BIND local zone records) |
| named_conf_path | required if --bind is specified | (str) path where named.conf lives, e.g. /etc/bind |
| ip | required | (str) public ip address of the machine |
| port | required | (int) port for wireguard to listen on |
| net | required | (str) vpn subnet with cidr notation |
| mtu | required | (int) mtu value for the interface (max is 1340 for tcp and 1460 for udp) |
| priv | optional | (str) private wireguard key (will be autogenerated if empty) |
| udp2raw_secret | optional | (str) upd2raw secret (will be autogenerated if empty) |
| udp2raw_port | required if proto == tcp | (int) port for upd2raw to listen on |
| extra_address | optional | (str) extra /32 to be appended to the Address line of the server Interface and to the AllowedIPs of the clients that opted in |
### clients
| key | necessity | description |
|---|---|---|
| name | required | (str) name for the interface |
| tcp | optional | (bool) if set to true, client will become associated with all servers that has proto set to tcp |
| udp2raw_log_path | required if tcp | (str) path to redirect udp2raw output to |
| bind | optional | (bool) if set to true, named.conf handling will be included in the config file generated |
| root_zone_file | required if bind | (str)path to the root zone file |
| android | optional | (bool) whether client is an android device |
| wgquick_path | required if android && tcp | (str) path to the wg-quick binary |
| upd2raw_path | required if android && tcp | (str) path to the udp2raw binary |
| append_extra | optional | (bool) append the server's extra_address (if it exists) to the AllowedIPs for the client |
| extra_allowed | optional | (str) extra srcip's to be accepted by the server from this client |
## example yaml
```yml
servers:
- name: wg0
  hostname: goodhostname
  named_conf_path: /etc/bind
  proto: udp
  ip: 1.1.1.1
  port: 51821
  net: 10.0.0.0/24
  mtu: 1420
  extra_address: 172.16.0.99/32
 
- name: wg0raw
  proto: tcp
  hostname: evenbetterhostname
  named_conf_path: /etc/bind
  ip: 2.2.2.2
  port: 51821
  net: 10.0.1.0/24
  mtu: 1340
  udp2raw_port: 6971

clients:
- name: linuxdesktop
  tcp: true
  bind: true
  root_zone_file: /var/named/zone/root-nov6
  udp2raw_log_path: /var/log/udp2raw.log
  append_extra: true
  extra_allowed:
    - 172.16.0.0/24

- name: androidphone
  tcp: true
  android: true
  udp2raw_log_path: ./udp2raw.log
  wgquick_path: /system/bin/wg-quick
  udp2raw_path: /data/data/com.termux/files/home/udp2raw

- name: vm
```
