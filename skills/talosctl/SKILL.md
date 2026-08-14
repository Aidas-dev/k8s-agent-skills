---
name: talosctl
description: Talos Linux CLI operations — cluster lifecycle, upgrades, reboots, node-level diagnosis, etcd maintenance, and troubleshooting with talosctl.
---

# talosctl — CLI Operations

For machine config YAML (documents, patching) see `talosconfig`. For Terraform provider see `talos-terraform`.

## ⚠️ Configuration Changes Require Dry-Run + Verification

**Every `apply-config` and `patch machineconfig` MUST be dry-run first, and the user MUST review/approve the output before anything is applied:**

```bash
# Dry-run first — always
talosctl patch machineconfig --nodes <ip> --patch @patch.yaml --dry-run
talosctl apply-config --nodes <ip> --file config.yaml --dry-run
```

Only after the user explicitly confirms the dry-run output, run without `--dry-run`.

## ⚠️ apply-config Needs the FULL Config File

`talosctl apply-config --file <config>` replaces the node's **entire** machine configuration with the contents of `<config>`. It is NOT a patch tool — providing a partial YAML wipes all unspecified settings.

- **Full-file changes** (install, network rewrite, secrets rotation) → `talosctl apply-config --file <full-config.yaml>`
- **Targeted changes** (one field) → `talosctl patch machineconfig --patch @patch.yaml`
- Get the current full config first: `talosctl get machineconfig v1alpha1 -o yaml`

## CLI Quick Reference

### Cluster Lifecycle

| Command | Purpose |
|---------|---------|
| `talosctl gen config <name> <endpoint> --with-secrets secrets.yaml` | Generate cluster configs (full config files) |
| `talosctl gen secrets --output-file secrets.yaml` | Generate secrets bundle |
| `talosctl apply-config --nodes <ip> --file config.yaml --insecure` | Apply FULL config to new node |
| `talosctl bootstrap --nodes <ip>` | Bootstrap etcd (once, first CP only) |
| `talosctl kubeconfig --nodes <ip> --force` | Get admin kubeconfig (overwrite existing) |
| `talosctl health --nodes <cps> --wait-timeout=10m` | Check cluster health |
| `talosctl version` | Show client + server versions |

### Maintenance

| Command | Purpose |
|---------|---------|
| `talosctl upgrade --image ghcr.io/siderolabs/installer:v1.13.8 --reboot-mode=default` | Upgrade Talos OS (drains node by default) |
| `talosctl upgrade --no-reboot` | Upgrade, defer reboot (replaces deprecated `--stage`) |
| `talosctl upgrade --drain=false` | Upgrade without draining (use with care) |
| `talosctl upgrade-k8s --to v1.36.2 --dry-run` | Upgrade Kubernetes (dry-run first) |
| `talosctl upgrade-k8s --to v1.36.2` | Upgrade Kubernetes |
| `talosctl rollback --nodes <ip>` | Rollback Talos to previous install |
| `talosctl reboot --mode=default` | Reboot node (default/force/powercycle) |
| `talosctl reset --graceful --reboot` | Reset node (removes from cluster) |
| `talosctl shutdown` | Shutdown node (cordon/drain first, `--force` to skip) |

### Config Changes

| Command | Purpose |
|---------|---------|
| `talosctl patch machineconfig --nodes <ip> --patch @patch.yaml --dry-run` | Patch specific fields (DRY-RUN FIRST) |
| `talosctl patch machineconfig --nodes <ip> --patch-file patch.yaml` | Patch from file |
| `talosctl apply-config --nodes <ip> --file <FULL-config.yaml> --dry-run` | Replace FULL config (DRY-RUN FIRST) |
| `talosctl apply-config --nodes <ip> --file cfg.yaml -p @patch.yaml` | Apply full config + on-the-fly patches |
| `talosctl get machineconfig v1alpha1 -o yaml` | Get current full config (with ID) |
| `talosctl edit machineconfig` | Edit config in default editor |
| `talosctl validate --mode metal --strict config.yaml` | Validate config offline (metal/cloud/container) |

**Patch flags:** `-p, --patch stringArray` (inline or `@file`), `--patch-file`. **Apply modes:** `-m, --mode auto|no-reboot|reboot|staged|try`.

## Node-Level Diagnosis

Extensive per-node troubleshooting. Add `--nodes <ip>` to every command (or `-n` for multiple).

### Services & Logs

```bash
talosctl service                       # all services + state
talosctl service <id>                  # single service status
talosctl service <id> start|stop|restart
talosctl logs <service> --tail 100     # service logs (kubelet, etcd, containerd, etc.)
talosctl logs <service> -f             # follow logs
talosctl restart <id>                  # restart a process (e.g. kubelet)
talosctl dmesg                         # kernel logs
talosctl dmesg -f                      # follow kernel logs
```

### Complete Resource Reference (v1.13.7 registry)

`talosctl get <type>` is the primary read API — Talos models ALL system state as resources. **All 167 types** (165 machinery + 2 COSI meta) below. Full names are `<Name>.<suffix>`; aliases are interchangeable. Not every type populates on every node: control-plane types (etcd, secrets, k8s controlplane ns) only on CP nodes, kubespan only when enabled, secrets/* are sensitive (talosctl redacts). This list is version-bound — run `talosctl get rd` to discover types added by extensions.

```bash
talosctl get rd                        # list ALL available resource types on this node
talosctl get <type> -o yaml            # full spec for any resource
talosctl get <type> <id>               # single resource by ID
talosctl get <type> -w                 # watch changes
```

**Common shortcuts:** `mc`/`machineconfig`, `rd`, `links`, `routes`, `addresses`, `resolvers`, `hostname`, `disks`, `cpu`, `members`, `services`/`svc`, `nodename`, `timestatus`, `mounts`.

#### meta (available on every node)

| Type | Aliases | Shows |
|------|---------|-------|
| `ResourceDefinitions` | `rd`, `rds`, `resourcedefinitions`, `api-resources` | Every registered type: aliases, print columns |
| `Namespaces` | `ns`, `namespaces` | All namespaces in runtime state |

#### block / storage (ns runtime)

| Type | Aliases | Shows |
|------|---------|-------|
| `Disks` | `disk`, `disks` | Physical disks: size, model, serial, transport, WWID |
| `BlockDevices` | `blockdevice`, `blockdevices` | Block devices (partitions): type, partition name |
| `DiscoveredVolumes` | `discoveredvolume`, `discoveredvolumes` | Volumes found on disks (install/bootstrap) |
| `VolumeStatuses` | `volumestatus`, `volumestatuses` | Volume state: type, phase, location, size |
| `VolumeConfigs` | `volumeconfig`, `volumeconfigs` | Desired volume configuration |
| `VolumeLifecycles` | `volumelifecycle`, `volumelifecycles` | Volume lifecycle signals |
| `SystemDisks` | `systemdisk`, `systemdisks` | Which disk is the Talos system disk |
| `MountStatuses` (block) | `mountstatus`, `mountstatuses` | Block-layer mounts: source, target, fs, volume |
| `MountRequests` | `mountrequest`, `mountrequests` | Pending mount requests |
| `VolumeMountRequests` | `volumemountrequest`, `volumemountrequests` | Volume mount requests |
| `VolumeMountStatuses` | `volumemountstatus`, `volumemountstatuses` | Volume mount state |
| `SwapStatuses` | `swap`, `swaps`, `swapstatus`, `swapstatuses` | Swap devices: size, used, priority |
| `BlockSymlinks` | `blocksymlink`, `blocksymlinks` | Symlinks into block device tree |
| `UserDiskConfigStatuses` | `userdiskconfigstatus`, `userdiskconfigstatuses` | User-disk config readiness |
| `ZswapStatuses` | `zswapstatus`, `zswapstatuses` | zswap stats: size, stored, written back |
| `DiscoveryRefreshRequests` | `discoveryrefreshrequest`, `discoveryrefreshrequests` | Re-run disk/volume discovery |
| `DiscoveryRefreshStatuses` | `discoveryrefreshstatus`, `discoveryrefreshstatuses` | Discovery refresh status |

#### cluster (ns cluster)

| Type | Aliases | Shows |
|------|---------|-------|
| `Identities` | `identity`, `identities` | Node cluster identity (ID) |
| `Members` | `member`, `members` | Cluster members: hostname, type, OS, addresses |
| `Affiliates` | `affiliate`, `affiliates` | Discovered affiliates / KubeSpan peers |
| `Infos` | `info`, `infos` | Cluster ID, cluster name |
| `DiscoveryConfigs` | `discoveryconfig`, `discoveryconfigs` | Discovery configuration |

#### config (ns config)

| Type | Aliases | Shows |
|------|---------|-------|
| `MachineConfigs` | `mc`, `mcs`, `machineconfig`, `machineconfigs` | Active/persistent machine config YAML |
| `MachineTypes` | `machinetype`, `machinetypes`, `mt`, `mts` | controlplane vs worker |

#### cri (ns cri)

| Type | Aliases | Shows |
|------|---------|-------|
| `ImageCacheConfigs` | `imagecacheconfig`, `imagecacheconfigs` | CRI image cache status |
| `SeccompProfiles` | `seccompprofile`, `seccompprofiles` | Seccomp profiles |
| `RegistryConfigs` | `registryconfig`, `registryconfigs`, `registries` | Container registry configuration |

#### etcd (ns etcd)

| Type | Aliases | Shows |
|------|---------|-------|
| `EtcdSpecs` | `etcdspec`, `etcdspecs` | etcd config: name, addresses |
| `EtcdMembers` | `etcdmember`, `etcdmembers` | etcd cluster members: member ID |
| `EtcdConfigs` | `etcdconfig`, `etcdconfigs` | etcd image config |
| `PKIStatuses` | `pkistatus`, `pkistatuses` | etcd PKI readiness: ready, secrets version |

#### files (ns files)

| Type | Aliases | Shows |
|------|---------|-------|
| `EtcFileSpecs` | `etcfilespec`, `etcfilespecs` | Desired /etc files |
| `EtcFileStatuses` | `etcfilestatus`, `etcfilestatuses` | Applied /etc file state |

#### hardware (ns hardware)

| Type | Aliases | Shows |
|------|---------|-------|
| `Processors` | `cpu`, `cpus`, `processor`, `processors` | CPU topology and features |
| `MemoryModules` | `ram`, `memorymodule`, `memorymodules` | RAM modules: manufacturer, model, size |
| `PCIDevices` | `device`, `devices`, `pcidevice`, `pcidevices` | PCI devices: class, vendor, product |
| `SystemInformations` | `systeminformation`, `systeminformations` | DMI: vendor, model, UUID, firmware |
| `PCRStatuses` | `pcrstatus`, `pcrstatuses` | TPM PCR status |
| `PCIDriverRebindConfigs` | `pcidriverrebindconfig`, `pcidriverrebindconfigs` | Requested PCI driver rebinds |
| `PCIDriverRebindStatuses` | `pcidriverrebinds`, `pcidriverrebindstatus`, `pcidriverrebindstatuses` | PCI driver rebind results |

#### k8s node-local (ns k8s)

| Type | Aliases | Shows |
|------|---------|-------|
| `Nodenames` | `nodename`, `nodename...` | Kubernetes node name |
| `NodeStatuses` | `nodestatus`, `nodestatuses` | Node ready, unschedulable |
| `NodeIPs` | `nodeip`, `nodeips` | Resolved node IPs |
| `NodeIPConfigs` | `nodeipconfig`, `nodeipconfigs` | Node IP selection config |
| `KubeletConfigs` | `kubeletconfig`, `kubeletconfigs` | Desired kubelet configuration |
| `KubeletSpecs` | `kubeletspec`, `kubeletspecs` | Kubelet image, args |
| `KubeletLifecycles` | `kubeletlifecycle`, `kubeletlifecycles` | Kubelet lifecycle flags |
| `KubeletKubeconfigs` | `kubeletkubeconfig`, `kubeletkubeconfigs` | Kubelet kubeconfig hash |
| `StaticPods` | `staticpod`, `staticpods` | Static pod definitions |
| `StaticPodStatuses` | `podstatus`, `staticpodstatus`, `staticpodstatuses` | Static pod readiness |
| `StaticPodServerStatuses` | `staticpodserverstatus`, `staticpodserverstatuses` | Static pod server status |
| `NodeLabelSpecs` | `nodelabelspec`, `nodelabelspecs` | Labels to apply to node |
| `NodeTaintSpecs` | `nodetaintspec`, `nodetaintspecs` | Taints to apply to node |
| `NodeAnnotationSpecs` | `nodeannotationspec`, `nodeannotationspecs` | Annotations to apply to node |
| `NodeCordonedSpecs` | `nodecordonedspec`, `nodecordonedspecs` | Node cordon state |
| `KubePrismConfigs` | `kubeprismconfig`, `kubeprismconfigs` | KubePrism LB config |
| `KubePrismEndpoints` | `kubeprismendpoint`, `kubeprismendpoints` | KubePrism backend endpoints |
| `KubePrismStatuses` | `kubeprismstatus`, `kubeprismstatuses` | KubePrism health data |

#### k8s control plane (ns controlplane)

| Type | Aliases | Shows |
|------|---------|-------|
| `Endpoints` | `endpoint`, `endpoints` | API server endpoints |
| `Manifests` | `manifest`, `manifests` | Rendered control-plane manifests |
| `ManifestStatuses` | `manifeststatus`, `manifeststatuses` | Manifest apply status |
| `BootstrapManifestsConfigs` | `bootstrapmanifestsconfig`, `bootstrapmanifestsconfigs` | Bootstrap manifests config |
| `ExtraManifestsConfigs` | `extramanifestsconfig`, `extramanifestsconfigs` | Extra manifest URLs/contents |
| `ConfigStatuses` | `configstatus`, `configstatuses` | Kubeconfig readiness |
| `SecretStatuses` | `secretstatus`, `secretstatuses` | Kubernetes secrets readiness |
| `APIServerConfigs` | `apiserverconfig`, `apiserverconfigs` | kube-apiserver config |
| `ControllerManagerConfigs` | `controllermanagerconfig`, `controllermanagerconfigs` | kube-controller-manager config |
| `SchedulerConfigs` | `schedulerconfig`, `schedulerconfigs` | kube-scheduler config |
| `AdmissionControlConfigs` | `admissioncontrolconfig`, `admissioncontrolconfigs` | Admission control plugin config |
| `AuditPolicyConfigs` | `auditpolicyconfig`, `auditpolicyconfigs` | Audit policy config |
| `AuthorizationConfigs` | `authorizationconfig`, `authorizationconfigs` | Authorization config |

#### kubeaccess (ns config)

| Type | Aliases | Shows |
|------|---------|-------|
| `KubernetesAccessConfigs` | `kubernetesaccessconfig`, `kubernetesaccessconfigs` | API access: roles, namespaces |

#### kubespan

| Type | Aliases | Shows |
|------|---------|-------|
| `KubeSpanConfigs` | `kubespanconfig`, `kubespanconfigs` | KubeSpan configuration |
| `KubeSpanIdentities` | `kubespanidentity`, `kubespanidentities` | Node identity: address, pubkey |
| `KubeSpanEndpoints` | `kubespanendpoint`, `kubespanendpoints` | WireGuard endpoints |
| `KubeSpanPeerSpecs` | `kubespanpeerspec`, `kubespanpeerspecs` | Desired peers |
| `KubeSpanPeerStatuses` | `kubespanpeerstatus`, `kubespanpeerstatuses` | Peer state: endpoint, RX, TX |

#### network (ns network)

| Type | Aliases | Shows |
|------|---------|-------|
| `LinkStatuses` | `link`, `links`, `linkstatus`, `linkstatuses` | Network interfaces: type, kind, MAC, oper state |
| `LinkSpecs` | `linkspec`, `linkspecs` | Desired link config |
| `LinkRefreshes` | `linkrefresh`, `linkrefreshes` | Refresh link state signal |
| `LinkAliasSpecs` | `linkaliasspec`, `linkaliasspecs` | Desired interface alias |
| `EthernetStatuses` | `ethtool`, `ethernetstatus`, `ethernetstatuses` | Ethtool info: link, speed |
| `EthernetSpecs` | `ethernetspec`, `ethernetspecs` | Desired ethtool settings |
| `AddressStatuses` | `address`, `addresses`, `addressstatus`, `addressstatuses` | IP addresses |
| `AddressSpecs` | `addressspec`, `addressspecs` | Desired addresses |
| `HardwareAddresses` | `hardwareaddress`, `hardwareaddresses` | Desired MAC addresses |
| `RouteStatuses` | `route`, `routes`, `routestatus`, `routestatuses` | Routes: destination, gateway, link |
| `RouteSpecs` | `routespec`, `routespecs` | Desired routes |
| `RoutingRuleStatuses` | `routingrule`, `routingrules`, `routingrulestatus`, `routingrulestatuses` | Policy routing rules |
| `RoutingRuleSpecs` | `routingrulespec`, `routingrulespecs` | Desired routing rules |
| `ResolverStatuses` | `resolver`, `resolvers`, `resolverstatus`, `resolverstatuses` | DNS resolvers, search domains |
| `ResolverSpecs` | `resolverspec`, `resolverspecs` | Desired resolvers |
| `HostnameStatuses` | `hostname`, `hostnamestatus`, `hostnamestatuses` | Hostname + domainname |
| `HostnameSpecs` | `hostnamespec`, `hostnamespecs` | Desired hostname |
| `TimeServerStatuses` | `timeserver`, `timeservers`, `timeserverstatus`, `timeserverstatuses` | NTP servers in use |
| `TimeServerSpecs` | `timeserverspec`, `timeserverspecs` | Desired NTP servers |
| `NodeAddresses` | `nodeaddress`, `nodeaddresses` | Node addresses + sort algorithm |
| `NodeAddressFilters` | `nodeaddressfilter`, `nodeaddressfilters` | Include/exclude subnet filters |
| `NodeAddressSortAlgorithms` | `nodeaddresssortalgorithm`, `nodeaddresssortalgorithms` | Address sort algorithm |
| `NfTablesChains` | `chain`, `chains`, `nftableschain`, `nftableschains` | nftables chains: type, hook, priority |
| `DNSUpstreams` | `dnsupstream`, `dnsupstreams` | DNS upstream resolvers: healthy |
| `DNSResolveCaches` | `dnsresolvecache`, `dnsresolvecaches` | Host DNS resolve cache |
| `HostDNSConfigs` | `hostdnsconfig`, `hostdnsconfigs` | Host DNS config: enabled |
| `ProbeStatuses` | `probe`, `probes`, `probestatus`, `probestatuses` | Connectivity probe results |
| `ProbeSpecs` | `probespec`, `probespecs` | Desired probes |
| `OperatorSpecs` | `operatorspec`, `operatorspecs` | Network operators (DHCP, VIP) |
| `PlatformConfigs` | `platformconfig`, `platformconfigs` | Cloud platform network config |
| `DeviceConfigSpecs` | `deviceconfigspec`, `deviceconfigspecs` | Network device config |
| `NetworkStatuses` | `netstatus`, `netstatuses`, `networkstatus`, `networkstatuses` | Overall network status |

#### perf (ns perf)

| Type | Aliases | Shows |
|------|---------|-------|
| `CPUStats` | `cpustat`, `cpustats` | Last CPU stats: user, system |
| `MemoryStats` | `memorystat`, `memorystats` | Last memory stats: used, total |

#### runtime (ns runtime)

| Type | Aliases | Shows |
|------|---------|-------|
| `MachineStatuses` | `machinestatus`, `machinestatuses` | Machine lifecycle: stage, ready |
| `MachineResetSignals` | `machineresetsignal`, `machineresetsignals` | Reset-in-progress signal |
| `Versions` | `version`, `versions` | Talos version status |
| `PlatformMetadatas` | `platformmetadata`, `platformmetadatas` | Cloud platform metadata |
| `KernelCmdlines` | `cmdline`, `kernelcmdline`, `kernelcmdlines` | Kernel command line |
| `KernelParamSpecs` | `kernelparamspec`, `kernelparamspecs` | Desired kernel params/sysctls |
| `KernelParamDefaultSpecs` | `kernelparamdefaultspec`, `kernelparamdefaultspecs` | Default kernel params |
| `KernelParamStatuses` | `sysctls`, `kernelparams`, `kernelparameters`, `kernelparamstatus`, `kernelparamstatuses` | Applied sysctl state |
| `KernelModuleSpecs` | `kernelmodulespec`, `kernelmodulespecs` | Kernel modules to load |
| `LoadedKernelModules` | `module`, `modules`, `loadedkernelmodule`, `loadedkernelmodules` | Loaded kernel modules |
| `MetaKeys` | `meta`, `metakey`, `metakeys` | META partition key/value pairs |
| `MetaLoads` | `metaload`, `metaloads` | META loaded marker |
| `UniqueMachineTokens` | `uniquemachinetoken`, `uniquemachinetokens` | Unique machine token |
| `Environments` | `env`, `environment`, `environments` | Environment variables |
| `MountStatuses` (runtime) | `mounts`, `mountstatus`, `mountstatuses` | Runtime mounts |
| `ServicePIDs` | `servicepid`, `servicepids` | Service → PID mapping |
| `KmsgLogConfigs` | `kmsglogconfig`, `kmsglogconfigs` | Kernel log streaming config |
| `OOMActions` | `oomaction`, `oomactions` | OOM action records |
| `BootedEntries` | `bootedentry`, `bootedentries` | Booted bootloader entry |
| `Diagnostics` | `diagnostic`, `diagnostics` | Diagnostic warnings |
| `SecurityStates` | `securitystate`, `securitystates` | Secureboot, UKI, SELinux state |
| `SBOMItems` | `sbomitem`, `sbomitems` | SBOM entries |
| `ExtensionStatuses` | `extensions`, `extensionstatus`, `extensionstatuses` | Installed system extensions |
| `ExtensionServiceConfigs` | `extensionserviceconfig`, `extensionserviceconfigs` | Extension service config |
| `ExtensionServiceConfigStatuses` | `extensionserviceconfigstatus`, `extensionserviceconfigstatuses` | Extension service config status |
| `MaintenanceServiceConfigs` | `maintenanceserviceconfig`, `maintenanceserviceconfigs` | Maintenance service config |
| `MaintenanceServiceRequests` | `maintenanceservicerequest`, `maintenanceservicerequests` | Maintenance service request |
| `APIServiceConfigs` | `apiserviceconfig`, `apiserviceconfigs` | Talos API service config |
| `EventSinkConfigs` | `eventsinkconfig`, `eventsinkconfigs` | Event sink (log shipping) config |
| `WatchdogTimerConfigs` | `watchdogtimerconfig`, `watchdogtimerconfigs` | Watchdog config: device, timeout |
| `WatchdogTimerStatuses` | `watchdogtimerstatus`, `watchdogtimerstatuses` | Watchdog status |
| `DevicesStatuses` | `devicesstatus`, `devicesstatuses` | Hardware devices readiness |

#### secrets (ns secrets, control-plane, sensitive)

| Type | Aliases | Shows |
|------|---------|-------|
| `OSRootSecrets` | `osrootsecret`, `osrootsecrets` | OS root CA & certs |
| `KubernetesRootSecrets` | `kubernetesrootsecret`, `kubernetesrootsecrets` | K8s root CA & SA keys |
| `KubernetesSecrets` | `kubernetessecret`, `kubernetessecrets` | K8s cert bundle |
| `KubernetesDynamicCerts` | `kubernetesdynamiccert`, `kubernetesdynamiccerts` | Dynamically issued K8s certs |
| `KubeletSecrets` | `kubeletsecret`, `kubeletsecrets` | Kubelet client cert bundle |
| `EtcdRootSecrets` | `etcdrootsecret`, `etcdrootsecrets` | etcd root CA & certs |
| `EtcdSecrets` | `etcdsecret`, `etcdsecrets` | etcd server/peer certs |
| `TrustdCertificates` | `trustdcertificate`, `trustdcertificates` | Trustd cert bundle |
| `ApiCertificates` | `apicertificate`, `apicertificates` | Talos API cert bundle |
| `MaintenanceRootSecrets` | `maintenancerootsecret`, `maintenancerootsecrets` | Maintenance-mode root secrets |
| `CertSANs` | `certsan`, `certsans` | SANs for API certs |
| `EncryptionSalts` | `encryptionsalt`, `encryptionsalts` | Encryption salt for secrets |

#### security (ns security)

| Type | Aliases | Shows |
|------|---------|-------|
| `TUFTrustedRoots` | `tuftrustedroot`, `tuftrustedroots` | TUF trusted root refresh |
| `ImageVerificationRules` | `imageverificationrule`, `imageverificationrules` | Image verification policy |

#### siderolink (ns config)

| Type | Aliases | Shows |
|------|---------|-------|
| `SiderolinkConfigs` | `siderolinkconfig`, `siderolinkconfigs` | SideroLink config |
| `SiderolinkTunnels` | `siderolinktunnel`, `siderolinktunnels` | SideroLink tunnel state |
| `SiderolinkStatuses` | `siderolinkstatus`, `siderolinkstatuses` | SideroLink connection status |

#### time (ns runtime)

| Type | Aliases | Shows |
|------|---------|-------|
| `TimeStatuses` | `timestatus`, `timestatuses` | NTP sync status |
| `AdjtimeStatuses` | `adjtimestatus`, `adjtimestatuses` | adjtimex state: offset, status |

#### v1alpha1 (ns runtime)

| Type | Aliases | Shows |
|------|---------|-------|
| `Services` | `svc`, `service`, `services` | Talos services: running, healthy |
| `AcquireConfigSpecs` | `acquireconfigspec`, `acquireconfigspecs` | Config acquisition signal |
| `AcquireConfigStatuses` | `acquireconfigstatus`, `acquireconfigstatuses` | Config acquired; boot proceeds |

### System Inspection

| Command | Purpose |
|---------|---------|
| `talosctl containers` | List system containers (`-k` for K8s namespace) |
| `talosctl processes` | List running processes (alias: ps) |
| `talosctl stats` | Container stats (CPU/mem per container) |
| `talosctl memory` | Memory usage (alias: free) |
| `talosctl cgroups --preset memory` | cgroupv2 usage (cpu/cpuset/io/memory/process/psi/swap) |
| `talosctl mounts` | Mount points |
| `talosctl list /var` | Directory listing (alias: ls) |
| `talosctl read /path` | Read a file (alias: cat) |
| `talosctl usage /var` | Disk usage (alias: du) |
| `talosctl copy /path .` | Copy data out of node |
| `talosctl image list` | Images in container runtime |
| `talosctl image pull <ref>` | Pull image into node |
| `talosctl time --check <ntp-server>` | Server time + NTP check (`--check server` form) |
| `talosctl events` | Stream runtime events (`--duration 1h` for history) |
| `talosctl inspect dependencies` | Controller-resource dependency graph |
| `talosctl meta write` | Write META partition keys |

### Network Diagnosis

```bash
talosctl netstat -t --listening        # TCP listening sockets (host)
talosctl netstat -a                    # all socket states
talosctl netstat -k                    # sockets used by K8s pods
talosctl netstat <namespace>/<pod>     # specific pod's connections
talosctl pcap -i eth0                  # live packet capture (tcpdump alias)
talosctl pcap -i eth0 --output file.pcap
```

### Diagnostics & Debug

| Command | Purpose |
|---------|---------|
| `talosctl dashboard` | Real-time cluster dashboard |
| `talosctl support` | Dump full debug archive (logs, configs, resources) |
| `talosctl debug` | Run a debug container on node |
| `talosctl version` | Version info (client + node) |
| `talosctl conformance kubernetes` | Run K8s conformance tests |

### etcd

| Command | Purpose |
|---------|---------|
| `talosctl etcd members` | List etcd cluster members |
| `talosctl etcd status` | etcd cluster member status |
| `talosctl etcd alarm list` | Check etcd alarms |
| `talosctl etcd alarm disarm` | Clear no-space alarms |
| `talosctl etcd snapshot snapshot.db` | Backup etcd |
| `talosctl etcd defrag --nodes <ip>` | Defrag etcd database |
| `talosctl etcd forfeit-leadership` | Move leader for maintenance |
| `talosctl etcd remove-member <member>` | Remove member from cluster |
| `talosctl etcd leave` | Make node leave etcd cluster |

## Apply Modes

| Mode | Behavior |
|------|----------|
| `auto` (default) | Immediate apply, reboot if needed |
| `no-reboot` | Apply without reboot |
| `reboot` | Always reboot to apply |
| `staged` | Apply on next reboot |
| `try` | Apply, rollback after timeout if not confirmed (default 1m) |

## Reboot Modes

| Mode | Behavior |
|------|----------|
| `default` | Normal reboot via kexec |
| `powercycle` | Skip kexec (cold boot, useful for BMC) |
| `force` | Skip graceful teardown (emergency only) |

## Common Patterns

### Bootstrap cluster

```bash
talosctl gen secrets --output-file secrets.yaml
talosctl gen config prod https://10.0.1.100:6443 \
  --with-secrets secrets.yaml \
  --config-patch-control-plane @network.yaml
talosctl apply-config --nodes 10.0.1.10 --file controlplane.yaml --insecure
talosctl bootstrap --nodes 10.0.1.10
talosctl apply-config --nodes 10.0.1.11 --file controlplane.yaml
talosctl kubeconfig --nodes 10.0.1.10
kubectl get nodes
```

### Upgrade (rolling CP)

```bash
# Sequential CP upgrade with verification
for node in cp1 cp2 cp3; do
  talosctl upgrade --nodes $node --image ghcr.io/siderolabs/installer:v1.13.8
  kubectl wait --for=condition=Ready node/$node --timeout=10m
  talosctl etcd members
  sleep 30
done
```

### Targeted config patch (dry-run first!)

```bash
# 1. Dry-run — show what changes WITHOUT applying
talosctl patch machineconfig --nodes cp1 --patch @patch.yaml --dry-run

# 2. After user confirms the diff, apply
talosctl patch machineconfig --nodes cp1 --patch @patch.yaml
```

### Full config replacement (dry-run first!)

```bash
# 1. Get current full config
talosctl get machineconfig v1alpha1 --nodes cp1 -o yaml > current.yaml

# 2. Edit, then dry-run the new full config
talosctl apply-config --nodes cp1 --file edited.yaml --dry-run

# 3. After user confirms, apply
talosctl apply-config --nodes cp1 --file edited.yaml
```

### Upgrade Kubernetes

```bash
talosctl upgrade-k8s --to v1.36.2 --dry-run
talosctl upgrade-k8s --to v1.36.2
```

### Node troubleshooting workflow

```bash
# 1. Services health
talosctl service --nodes <ip>

# 2. Failed service logs
talosctl logs <failing-service> --tail 100 --nodes <ip>

# 3. Kernel issues
talosctl dmesg --nodes <ip>

# 4. Resources (disk/CPU/mem)
talosctl memory --nodes <ip>
talosctl stats --nodes <ip>
talosctl get disks --nodes <ip>           # hardware disk inventory
talosctl get blockdevices --nodes <ip>    # block devices + partitions
talosctl get disks sda -o yaml --nodes <ip>  # single disk detail
talosctl usage /var --nodes <ip>          # filesystem-level usage

# 5. Network
talosctl get links --nodes <ip>
talosctl netstat -t --listening --nodes <ip>

# 6. Full archive for support
talosctl support --nodes <ip>
```

### etcd maintenance

```bash
talosctl etcd snapshot /backup/etcd-$(date +%Y%m%d).snapshot
talosctl etcd defrag --nodes cp1
talosctl etcd alarm list
talosctl etcd alarm disarm       # Clear no-space alarms
talosctl etcd forfeit-leadership # Move leader for maintenance
```

## Common Mistakes

- **Bootstrap more than once** — Only one `talosctl bootstrap` per cluster lifetime. Running it again creates split-brain.
- **Losing secrets** — Always `--with-secrets secrets.yaml` during `gen config`. Without secrets, you cannot add nodes, rotate certs, or recover. Store encrypted in Vault/SOPS.
- **Upgrade all CP simultaneously** — Sequential only. etcd needs quorum. One node at a time, verify health between each.
- **`talosctl apply-config` with a partial file** — apply-config REPLACES the whole config. Partial YAML wipes unspecified settings. Use `talosctl patch machineconfig` for targeted changes.
- **Skipping `--dry-run`** — Every config change must be dry-run first and reviewed by the user before applying.
- **`talosctl get disks` vs `talosctl disks`** — `get disks` uses the `disks.block.talos.dev` resource. `talosctl disks` is a separate command with different output.
- **`--stage`/`--preserve` flags deprecated** — Legacy flags in v1.13+, removed in Talos 1.18. Use `--no-reboot` instead of `--stage`.
- **Apply config to all nodes at once** — For CP changes, use sequential apply with health checks between nodes (etcd quorum).
- **`talosctl reset` without `--graceful=false`** — By default `--graceful=true` which cordons/drains and removes from etcd. Set `--graceful=false --reboot` for hard reset.
- **`talosctl get machineconfig` without ID** — Needs the resource ID: `talosctl get machineconfig v1alpha1`.
- **Wrong diagnostics commands** — `interfaces`/`routes` are NOT talosctl commands. Use `talosctl get links` / `talosctl get routes`.
