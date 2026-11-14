# IPTables Firewall Simulation Lab

A three-zone perimeter firewall built with Containerlab, Alpine Linux and iptables.

## Topology

| Zone | Subnet | Hosts |
|------|--------|-------|
| LAN (trusted) | 10.1.1.0/24 | lan-client `10.1.1.10` |
| DMZ | 192.168.10.0/24 | dmz-server (nginx) `192.168.10.10` |
| WAN (untrusted) | 172.16.1.0/24 | wan-client `172.16.1.100`, wan-attacker `172.16.1.200` |

The `router` node is the firewall: `eth1` = LAN (10.1.1.1), `eth2` = DMZ (192.168.10.1), `eth3` = WAN (172.16.1.1). A small `wan-switch` node (Linux bridge) puts the router, wan-client and wan-attacker on one WAN segment.

```
lan-client --eth1-- [eth1] router [eth3] --- wan-switch --- wan-client
                          [eth2]                  |
                             |                    +--------- wan-attacker
                        dmz-server
```

## Deploy / Destroy

Requires Docker and Containerlab. Nodes download packages with `apk`, so the Docker management network needs internet access.

```bash
sudo clab deploy -t lab.clab.yml
sudo clab destroy -t lab.clab.yml
```

## Firewall policy (`config/iptables.rules`)

The file is bind-mounted into the router and loaded with `iptables-restore`.

1. **Stateful tracking**: ESTABLISHED/RELATED accepted on INPUT and FORWARD.
2. **Threat mitigation**: all new traffic from `172.16.1.200` is dropped (INPUT and FORWARD).
3. **LAN policy**: `eth1` to `eth2` and `eth1` to `eth3` fully allowed.
4. **DMZ policy**: `eth3` to `192.168.10.10` allowed only on tcp/80 and tcp/443.
5. **Default deny**: `FORWARD` policy is `DROP` (so is `INPUT`).

## Verification

```bash
# Test 1: expect HTTP/1.1 200 OK
docker exec -it clab-iptables-firewall-lab-wan-client curl -I http://192.168.10.10

# Test 2: expect timeout / failure
docker exec -it clab-iptables-firewall-lab-wan-client nc -zv -w 3 192.168.10.10 22

# Test 3: expect timeout
docker exec -it clab-iptables-firewall-lab-wan-attacker curl --connect-timeout 3 http://192.168.10.10

# Test 4: expect 100% packet loss
docker exec -it clab-iptables-firewall-lab-wan-client ping -c 3 10.1.1.10

# Test 5: expect HTTP/1.1 200 OK
docker exec -it clab-iptables-firewall-lab-lan-client curl -I http://192.168.10.10
```

Inspect the live rules and counters with:

```bash
docker exec clab-iptables-firewall-lab-router iptables -L -n -v
```
