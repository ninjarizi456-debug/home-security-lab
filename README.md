
## Segmented Cybersecurity Home Lab

A small, segmented lab environment built in VirtualBox for hands-on security practice. Attacker, gateway, and monitoring roles run as separate VMs connected through an internal network, rather than everything sitting on one flat host-only network.

### Architecture

```
[ Kali Linux ]  ---\
  (attacker)         \
                       [ OPNsense ]  ---  [ Internet / NAT ]
                       (gateway)
                      /
[ Wazuh SIEM ]  ------
  (monitor)
```

All three VMs sit on a shared VirtualBox Internal Network (`range-lan`, `192.168.1.0/24`). OPNsense sits between that internal network and NAT, acting as the gateway and (eventually) the firewall boundary between segments.

| VM | Role | IP on range-lan |
|---|---|---|
| Kali Linux | Attacker | 192.168.1.51 |
| OPNsense | Gateway | 192.168.1.1 |
| Wazuh (Ubuntu Server) | Monitoring / SIEM | 192.168.1.60 |
| Victim machine | Target | *rebuild in progress* |

### Stack

- **VirtualBox** for virtualization
- **Kali Linux** as the attacker platform
- **OPNsense** as the gateway/firewall
- **Wazuh 4.14** (indexer, manager, dashboard, Filebeat) as the SIEM/monitoring layer, self-hosted on Ubuntu Server 22.04

### Build notes and troubleshooting log

**Host resource constraints.** The host machine has 8GB of RAM total. Running multiple VMs simultaneously during install (especially the Wazuh indexer, which is OpenSearch-based) caused repeated kernel soft lockups. Confirmed this by checking host RAM usage during the failure and finding it consistently above 90%. Resolution: run at most one or two VMs at a time during heavy install/config steps, and keep individual VM RAM allocations close to the documented minimums rather than generous overallocation.

**OPNsense WAN/LAN interface swap.** After initial install, LAN traffic wasn't reaching Kali despite matching Internal Network names on both VMs and confirmed connectivity at the VirtualBox hypervisor level (`VBoxManage showvminfo`). Diagnosed by comparing each NIC's MAC address (from `VBoxManage showvminfo`) against what OPNsense's console reported for `em0`/`em1`. Found WAN and LAN had been assigned to the opposite physical adapters during initial setup. Fixed via the OPNsense console's "Assign interfaces" option, manually specifying `em0` as WAN and `em1` as LAN by MAC address rather than relying on auto-detection.

**Wazuh dashboard authentication failure.** After a Wazuh install that included a mid-install soft lockup, the admin password listed in `wazuh-install-files.tar` no longer matched the indexer's internal user database, returning `Unauthorized` even via direct `curl` auth against the indexer API (ruling out a browser/UI issue). Resolved by running Wazuh's official password reset tool:
```bash
sudo bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -a -au admin
```
followed by a restart of `wazuh-indexer`, `wazuh-manager`, `wazuh-dashboard`, and `filebeat` to apply the new credentials.

**Accidental disk deletion.** During cleanup of orphaned VM files, deleted a `.vdi` that was still actively referenced by a running VM (identified after the fact via `VBoxManage showvminfo <vm> | findstr vdi`). Lesson: before deleting any `.vdi` that appears orphaned, confirm no registered VM references it, rather than assuming a missing `.vbox` file in the same folder means it's unused.

### Status

- [x] Kali Linux installed and networked
- [x] OPNsense installed, WAN/LAN correctly assigned
- [x] Wazuh installed (indexer, manager, dashboard, Filebeat) and accessible from Kali
- [ ] Victim machine rebuild
- [ ] Wazuh agent deployed on victim machine
- [ ] Network segmentation into separate attacker/victim/monitor zones with OPNsense firewall rules between them
