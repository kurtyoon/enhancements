---
sep: 33
title: "Cilium CNI Support"
status: draft
authors: [kurtyoon]
created: 2026-10-13
components: [submariner, submariner-operator, subctl, shipyard]
repos:
  - submariner-io/submariner
  - submariner-io/submariner-operator
  - submariner-io/subctl
  - submariner-io/shipyard
  - submariner-io/submariner-website
---

# Cilium CNI Support

Tracking issue: [submariner#3168](https://github.com/submariner-io/submariner/issues/3168).
Earlier design and prototype by sanek9: [submariner#4105](https://github.com/submariner-io/submariner/issues/4105),
[submariner#4106](https://github.com/submariner-io/submariner/pull/4106),
[submariner-operator#4146](https://github.com/submariner-io/submariner-operator/pull/4146),
[subctl#1835](https://github.com/submariner-io/subctl/pull/1835).

## Summary

This proposal adds support for clusters that run Cilium with kube-proxy replacement, BPF masquerading and
eBPF host routing. Today such a cluster is treated as a `generic` network plugin and cross-cluster pod
traffic fails.

The proposal is smaller than the two approaches discussed on #4106. It starts with two phases agreed in
review. Phase 1 adds a route-agent handler that keeps remote cluster CIDRs in Cilium's ip-masq-agent list.
The handler is loaded only when a field in the `submariner-global` ConfigMap is set, and no existing handler
changes. Phase 2 adds a Cilium deployment to Shipyard and a CI job. The target design also has the operator's
network discovery detect Cilium, which replaces the manual field; when that is implemented is decided after
the first two phases.

ClusterIP exports hosted on a Cilium cluster are a documented limitation. Removing it needs a change outside
the CNI integration, which I will propose separately for Globalnet clusters.

## Motivation

Issue #3168 has been open since 2024 and the maintainers have asked for a contributor more than once. At
least one user moved to Calico because of it
([comment](https://github.com/submariner-io/submariner/issues/4105#issuecomment-5913719417)), and the same
user, jdaln, has since offered to help with an implementation on #3168 and #4105. In November 2026 yboaron
described the masquerading conflict on #4247 as a known limitation and said that contributions for Cilium
support are welcome.

Cilium can be configured by hand so that pod traffic to another cluster is not masqueraded. The guide yboaron
points to on #4247 does it on Cilium 1.16 with the chart defaults, so with kube-proxy still running and no
BPF masquerading, by setting `enable-ipv4-masquerade` to false, after which `subctl verify` passed. That
switches masquerading off for every destination, which a cluster that relies on node SNAT for pod egress
cannot do, and the kube-proxy setup it covers is not the one this proposal is about. With kube-proxy
replacement and BPF masquerading, every manual variant needs the other clusters' ranges in advance: an
ip-masq-agent list that names them, an `ipv4-native-routing-cidr` wide enough to contain them all, or BPF
masquerading turned off, which drops Cilium back to legacy host routing on every node (see Alternatives).
Those ranges change whenever a cluster joins or leaves the clusterset, and with Globalnet they are allocated
at join time. The setting lives on every Cilium cluster in the set, so each change is an edit on each of
them, and a missed edit shows up as pod traffic leaving with a node address. The handler in this proposal
keeps the ip-masq-agent list current from the `Endpoint` objects route-agent already watches, which is the
one part a document cannot do. The prerequisites that do not change at runtime stay as documentation.

The #4105/#4106 attempt stalled on a design question. Submariner could impersonate a ClusterMesh peer,
program Cilium's BPF maps, or do something smaller. The maintainers asked for a proposal that compares
these options and says what the project would have to maintain.

### Why the generic path fails

The generic route-agent path assumes that pod traffic goes through the node's routing table and netfilter
hooks. With eBPF host routing that is no longer true for traffic from pods. I checked the following against
Cilium 1.20.2 and reproduced each of them on kind.

1. A pod packet to a remote cluster is resolved by a FIB lookup in `bpf_lxc` and redirected straight to
   the egress device (`bpf/bpf_lxc.c:1343-1372`). Host routes and ip rules are still honoured, so
   Submariner's routes work. The node's netfilter rules, MSS clamp and conntrack never see the packet.
2. A remote CIDR resolves to the `world` identity. BPF masquerading rewrites world-bound pod traffic on any
   device that runs Cilium's `cil_to_netdev` program, unless the destination is in the native-routing CIDR
   or the ip-masq-agent list (`bpf/lib/nat.h:639-787`). Submariner's netfilter rules cannot pre-empt it.
3. Cilium's device auto-detection picks up `vx-submariner`
   (`pkg/datapath/linux/devices_controller.go:624-713`). On kind, every node added it within seconds of the
   device appearing, and pod traffic left with the node's 240.x VTEP address. This is the "240.x source"
   symptom reported on #4106.
4. Traffic that reaches the gateway from `vx-submariner` or from the cable goes through the host stack, so
   the gateway's netfilter rules, including Globalnet, keep working for pods on other nodes.

Submariner adds two gaps of its own. The generic discoverer reads the pod CIDR from `--cluster-cidr`
flags (`pkg/discovery/network/generic.go:174-199`), which may not exist with kube-proxy replacement and
cluster-pool IPAM. And service discovery hands clients a remote ClusterIP, or with Globalnet the global IP
of an `externalIPs` Service, leaving the translation to kube-proxy on the receiving gateway.

## Scope and Support

Each status is one of Supported, Not Supported, Not Tested and Not Applicable. Supported means it was run on
kind and passed. Not Supported means it fails or is outside this proposal. Not Tested means it is expected to
work but has no run behind it. Not Applicable marks a connectivity type that does not exist with Globalnet.

### Supported configurations

Every row assumes IPv4, kube-proxy replacement, BPF masquerading and the settings under Prerequisites.

| Cilium configuration | Status |
|---|---|
| Tunnel routing, Kubernetes host-scope IPAM | Supported |
| Tunnel routing, cluster-pool IPAM | Supported |
| Native routing, Kubernetes host-scope IPAM | Supported |
| Native routing, cluster-pool IPAM | Supported |
| kube-proxy kept, masquerading in iptables | Not Supported |
| IPv6 or dual-stack | Not Supported |
| CNI chaining | Not Supported |
| Managed Cilium distributions (GKE Dataplane V2, Azure CNI Powered by Cilium) | Not Supported |
| OpenShift with Cilium | Not Supported |
| Cilium IPsec encryption together with the libreswan cable | Not Supported |

The four supported rows were run with the libreswan cable. WireGuard and the VXLAN cable are Supported in
tunnel mode with Kubernetes IPAM and Not Tested in the other three rows. On cluster-pool IPAM the user passes
the pod CIDR with `--clustercidr` at join until discovery reads it.

- **IPv6:** the handler writes IPv4 CIDRs and requires `enable-ipv4-masquerade`. IPv6 BPF masquerading is beta
  in Cilium, and IPv6 is left for later.
- **kube-proxy with iptables masquerading:** such a cluster keeps today's generic path, and the handler stays
  idle there, because Cilium's ip-masq-agent exists only in the BPF implementation. The documentation points
  it to the existing options: an `ipv4-native-routing-cidr` that covers the remote ranges,
  `egress-masquerade-interfaces` limited to the uplink, which does not help with libreswan, or masquerading
  turned off.
- **OpenShift:** the operator reports the plugin name `Cilium` there, and route-agent's kube-proxy handler
  does not register for that name, so the cluster gets no Submariner routes. It was not run, for lack of an
  OpenShift environment.

### Supported connectivity

The table applies to a Cilium cluster in a supported configuration.

| Connectivity | Without Globalnet | With Globalnet |
|---|---|---|
| **Pod to ClusterIP service export, backends on a Cilium cluster** | **Not Supported** | **Not Supported** |
| Pod to remote pod IP | Supported | Not Applicable |
| Pod to headless service export | Supported | Supported |
| Pod IP or headless export, from or to a pod-network pod on a gateway node | Supported | Supported with the identity-mark setting |
| Pod to ClusterIP service export, backends on a cluster with kube-proxy | Supported | Not Tested |
| Service discovery with `clusterset.local` names | Supported | Supported |

- **ClusterIP exports:** an exported ClusterIP service whose backends run on a Cilium cluster is not reachable
  from another cluster, with or without Globalnet. With kube-proxy replacement, a request that arrives through
  the cable is not translated on the gateway, so it does not reach a backend, while service discovery still
  answers with the ClusterIP. The limitation depends on the cluster that hosts the backends, not on the
  client: a client on a Cilium cluster reaches a ClusterIP export hosted on a cluster with kube-proxy.
  Headless exports are not affected and are the workaround. For Globalnet clusters, a follow-up proposal
  removes the limitation (see Follow-ups).
- **Pods on a gateway node:** the row is about pods on the pod network. Host-network pods, which include
  Submariner's gateway, route-agent and globalnet pods, are not affected. With Globalnet, pods there whose
  traffic goes through Cilium's L7 or DNS proxy, and clusters with endpoint routes enabled, are Not Tested.
- **Gateway failover:** Supported without Globalnet, where a label change between two gateway nodes lost about
  4 s of pod to pod traffic. Not Tested with Globalnet.
- **Network policy:** Not Tested. Cross-cluster traffic has the `world` identity, so only CIDR rules can match
  it.

### Prerequisites

These are the settings the user makes on a Cilium cluster. The Datapath section says why each is needed.

- **ip-masq-agent with an explicit list.** `ipMasqAgent.enabled=true` and
  `ipMasqAgent.config.nonMasqueradeCIDRs` holding this cluster's own pod and service CIDRs, plus `240.0.0.0/4`
  for Submariner's VTEP addresses. The user sets them once, at install. An explicit list replaces Cilium's
  built-in one, so it has to name the local ranges, and the handler does not create the ConfigMap. The remote
  clusters' CIDRs are not maintained by hand: the handler adds them. The default Globalnet range `242.0.0.0/8`
  is inside `240.0.0.0/4`, so with the default range the handler has nothing to add. Destinations outside the
  list are masqueraded as before.
- **Legacy host routing on gateway nodes.** A `CiliumNodeConfig` with `enable-host-legacy-routing: "true"` and
  `nodeSelector: submariner.io/gateway=true`. The setting applies when the Cilium agent starts, so every node
  that can become a gateway must be labelled before Cilium starts there, and labelling a node later needs an
  agent restart.
- **Identity mark off on gateway nodes, with Globalnet.** `enable-identity-mark: "false"` in the same
  `CiliumNodeConfig`. Without it, some pods on a gateway node lose all their egress. A cluster without
  Globalnet does not need it.

Device pinning is optional when the list is used. A pinned device set such as `devices: "eth+"` keeps
Cilium's programs off Submariner's devices: `vx-submariner` on every node, and on gateway nodes the cable
device, `vxlan-tunnel` with the VXLAN cable and `submariner` with WireGuard. libreswan has no device of its
own. Cilium masquerades on every device it manages, and without pinning it manages these as well, which is
why the list is needed. The supported rows were run without pinning.

One set of values that meets them, taken from the kind runs:

```yaml
# Helm values for Cilium
kubeProxyReplacement: true
bpf:
  masquerade: true
ipMasqAgent:
  enabled: true
  config:
    masqLinkLocal: false
    nonMasqueradeCIDRs:
      - 10.1.0.0/16    # this cluster's pod CIDR
      - 10.101.0.0/16  # this cluster's service CIDR
      - 240.0.0.0/4    # Submariner VTEP addresses
---
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  name: submariner-gateway
  namespace: kube-system
spec:
  nodeSelector:
    matchLabels:
      submariner.io/gateway: "true"
  defaults:
    enable-host-legacy-routing: "true"
    enable-identity-mark: "false"  # only with Globalnet
```

If the Cilium chart is applied server-side, as Helm 4 does, a later upgrade conflicts with the handler on the
list field. The handler section describes the conflict and how to avoid it.

To verify, in the Cilium agent of a gateway node:

```bash
cilium-dbg status --verbose | grep -E '^(Routing|Masquerading)'
# Routing:                Network: Tunnel [vxlan]   Host: Legacy
# Masquerading:           BPF (ip-masq-agent)   [eth0, vx-submariner]   10.1.0.0/24  [IPv4: Enabled, IPv6: Disabled]
cilium-dbg bpf ipmasq list
# the list in effect; the remote CIDRs appear once the clusters are connected
```

`Host: Legacy` is the gateway-node setting, and `BPF (ip-masq-agent)` shows that the list is in use. The
device list includes `vx-submariner` unless the devices are pinned.

Some settings apply only if the cluster uses the feature:

- If it has a default-deny `CiliumNetworkPolicy`, add `world` or CIDR rules for cross-cluster traffic.
- If a `CiliumEgressGatewayPolicy` covers a remote CIDR, list that CIDR in its `excludedCIDRs`.
- If it uses the Cilium host firewall, allow UDP 4800 between nodes and the cable ports from remote gateways
  in node policy.
- If it uses Cilium's own IPsec encryption, Cilium drops raw ESP from sources that are not Cilium nodes, which
  includes a remote Submariner gateway. libreswan then needs forced UDP encapsulation (`--force-udp-encaps`,
  not verified), or another cable driver.

### Cilium versions

Cilium 1.20 is Supported. Cilium 1.16 to 1.19 are Not Tested. Versions before 1.16 are Not Supported, because
the gateway-node setting needs `CiliumNodeConfig` (`cilium.io/v2`), which exists since 1.16.

## Design

### Datapath

This walk-through supports two claims the rest of the proposal depends on. On a worker node, the only thing
Submariner needs from Cilium is that traffic to a remote CIDR is not masqueraded. And every NAT in the design
stays in netfilter on the active gateway, which both directions of a flow pass through.

Cluster A has pod CIDR 10.1.0.0/16 and cluster B has 10.2.0.0/16. `W` is a worker and `G` the active
gateway. "Stack" means the host network namespace with its netfilter hooks; "BPF" means a Cilium program
handled the hop.

| Hop | Request | Reply |
|---|---|---|
| `W_A` pod | BPF: FIB lookup, route `10.2.0.0/16 via 240.x dev vx-submariner` | BPF: overlay decap to the pod |
| `W_A` `vx-submariner` | VXLAN encap, no SNAT (10.2.0.0/16 is not masqueraded) | |
| `G_A` | stack: ip rule 150, table 150, MSS clamp, cable | stack: decap, MSS, route via `cilium_host` |
| `G_B` | stack: decap, MSS, `cilium_host`, then BPF overlay | stack: table 150, MSS, cable |
| `W_B` pod | BPF: overlay decap, source identity `world` | BPF: FIB to `vx-submariner`, no SNAT |

The table shows tunnel mode. With native routing, the hops between `G_B` and `W_B` are plain routes instead
of Cilium's overlay, and the rest is the same.

Packets reach the active gateway from a device, not from a local pod, so both directions of a flow go through
its stack. The [appendix](https://gist.github.com/kurtyoon/366ac0e42c60dfbd6b605fa030ffa382) has hop tables
for Globalnet egress and headless ingress.

Remote CIDRs must not be masqueraded on any device that carries them. This is the one hard requirement on
the worker side, and there are two ways to meet it. The recommended way is ip-masq-agent with an explicit
list, which the handler keeps current. The other way is a pinned device set, so that no Submariner device
runs Cilium's programs. It only holds while the setting stays pinned, because Cilium attaches to devices at
runtime and `vx-submariner` is created after the agent starts. Pinning is also useful next to the list:
without it Cilium takes `vx-submariner`'s address as a NodePort address and applies host-firewall policy to
it.

Legacy host routing is only about pod-network pods that run on a gateway node. Their egress is redirected in
BPF along table 150 and skips the gateway's netfilter hooks. With libreswan this also skips the IPsec output
policy and the packet is dropped. With Globalnet, on any cable, it skips the Globalnet SNAT, and a backend's
reply skips the DNAT reversal. WireGuard without Globalnet happens to work. A host-network pod sends from the
host namespace and already goes through the stack. A `CiliumNodeConfig` that selects the gateway nodes fixed
every gateway-node case on kind. Keeping workloads off gateway nodes with a taint is an untested alternative.

Globalnet adds one more setting on gateway nodes. When Cilium passes pod traffic to the stack, it writes the
pod's identity into the upper 16 bits of the packet mark (`bpf/bpf_lxc.c:1204`). Globalnet's egress rules
match `0xC0000` in the same mark to find traffic bound for a remote cluster, and they do not clear those bits
first. With legacy host routing, a gateway-node pod whose identity has both bits set therefore has every new
connection SNATed to the cluster's global egress IP, including connections inside its own cluster. With
`enable-identity-mark: "false"` Cilium does not write the identity on that path and looks it up by source
address instead. Traffic that Cilium's L7 or DNS proxy forwards still carries the identity in the mark.

### Discovery (submariner-operator)

The operator's network discovery detects Cilium, so that the user does not pass `--clustercidr` at join or
name the Cilium namespace. Until that is implemented, a Cilium cluster is discovered as `generic`, and the
Implementation Plan says what the user sets in the meantime.

A new `pkg/discovery/network/cilium.go` runs after the existing discoverers and before the generic
fall-through. A Calico cluster that also carries a `cilium` DaemonSet, during a migration or in chaining,
therefore stays Calico and keeps its IPPool handler. Detection requires a single Cilium agent DaemonSet that
has ready pods and is scheduled on every Linux node, no marker of a managed distribution (GKE Dataplane V2,
Azure CNI Powered by Cilium) and no CNI chaining mode other than `portmap`. Otherwise the cluster stays
`generic`, and a log line says which condition failed. The Cilium namespace is the one that DaemonSet runs
in.

Discovery reports the pod and service CIDRs as the other discoverers do. The service CIDR keeps coming from
the existing generic sources, which do not depend on the CNI. The pod CIDR comes from the same sources, and
when they are empty on cluster-pool IPAM, from `cluster-pool-ipv4-cidr` in `cilium-config`.

A detected cluster is reported under the plugin name `cilium`, the way the other discoverers report their
CNI, and the handler and the diagnosis go by that name. The name has to be added to
`cni.GetNetworkPlugins()` in submariner, so that the kube-proxy handler still registers, and to
`supportedNetworkPlugins` and `networkPluginCNIType` in subctl, so that diagnose and gather treat the cluster
as they treat a `generic` one today.

OpenShift with Cilium stays out of scope. The OpenShift4 discoverer runs first and reports `Cilium` from
`spec.networkType`, and this proposal does not change that.

### route-agent Cilium handler

A new handler in `pkg/routeagent_driver/handlers/cilium` keeps every remote cluster CIDR in the
`nonMasqueradeCIDRs` list of Cilium's `ip-masq-agent` ConfigMap. It is active on a cluster where Cilium is
detected. It is additive, like the Calico IPPool handler: no existing handler changes. It uses the Kubernetes
client only, and it does not touch ipcache, `cilium-config`, `CiliumNode` or any BPF map.

It needs one new rule on the route-agent ClusterRole: `watch` and `update` on `configmaps` with
`resourceNames: [ip-masq-agent]`. The role already has `get` and `list` on ConfigMaps.

Behaviour:

- **Remote `Endpoint` created or updated:** add the missing remote CIDRs after the user's entries. The remote
  CIDRs are the IPv4 subnets of every remote `Endpoint`, recomputed from all of them each time.
- **Remote `Endpoint` updated or deleted, and a five minute resync:** remove the entries the handler added
  that are no longer wanted.
- **`ip-masq-agent` ConfigMap replaced by another writer:** add the remote CIDRs again. A chart upgrade that
  renders the list again is the usual case.
- **Gateway failover:** only the active gateway writes. A gateway that steps down removes nothing, since that
  would open a masquerade window.
- **Uninstall:** remove the entries the handler added and its two annotations.
- **Explicit list or Cilium setting missing:** log a warning and write nothing. The settings are
  `enable-ip-masq-agent`, `enable-bpf-masquerade` and `enable-ipv4-masquerade` in `cilium-config`.

The handler tells its entries from the user's by two annotations on the `ip-masq-agent` ConfigMap in the
Cilium namespace. `submariner.io/non-masquerade-cidrs` lists what it added, and entries not named there are
never touched. `submariner.io/non-masquerade-config-checksum` holds a checksum of the config it wrote. When
that no longer matches, another writer replaced the config, and the handler stops treating the listed entries
as its own. Entries are not removed on every event, because after a restart the remote Endpoints arrive one
at a time and removing then would briefly drop subnets that are still in use. The handler does not create the
ConfigMap, because a `create` permission cannot be limited by resource name, and a missing ConfigMap or
permission is logged and never fails `Init`.

An explicit list is required. With ip-masq-agent enabled and no ConfigMap, Cilium uses a built-in list that
covers all of RFC 1918 and the reserved ranges (`pkg/ipmasq/ipmasq.go:26-45`), which stops masquerading
toward every private address outside the cluster. On kind that broke pod connections to the broker API
server. A change to the list reaches Cilium after the kubelet ConfigMap sync, about a minute.

Known limitations:

- The list is one string field, `data.config`, with two writers. Helm 4 applies charts server-side, and on
  kind a `helm upgrade` of Cilium with unchanged values failed with a field conflict on that field between
  the managers `helm` and `submariner-route-agent`. It went through with `--force-conflicts`, after which the
  handler added the remote CIDRs back and traffic kept passing. Putting the remote CIDRs in the Helm values
  avoids the conflict, because the handler then has nothing to write. Helm 3, Argo CD and Flux were not run.
- If Cilium support is turned off before uninstall, the entries stay, because the handler is no longer loaded
  to remove them.
- If the ClusterRole rule is missing, the handler only logs the error.

### Globalnet

Globalnet needs no change of its own. Egress through the cluster and namespace `GlobalEgressIP`s,
headless-service ingress and the gateway health check are netfilter on the gateway, and both directions pass
through it. The pod-selector `GlobalEgressIP` uses the same path and was not run. ClusterIP exports are the
exception; see Scope and Support.

### Policy and identity

Cross-cluster traffic keeps the `world` identity, including Globalnet-translated traffic. No Cilium identity
is assigned to a remote cluster, so identity-based policy cannot select it. Giving remote clusters a Cilium
identity is what the ClusterMesh-shaped publisher would add; see Follow-ups.

### Post-deployment diagnosis

Once discovery detects Cilium, a deployed cluster can be checked against the prerequisites. `subctl diagnose
cni` is where CNI validation lives, and it can cover Cilium the way it covers OVN-Kubernetes. The diagnosis
could also live in [submariner-diagnostics](https://github.com/submariner-io/submariner-diagnostics), which
needs no `subctl` change. Which of the two is decided when it is written. `subctl join` gets no Cilium check.
In either place it covers:

- that ip-masq-agent is enabled with a ConfigMap and every remote CIDR is in the effective list, or that the
  device set is pinned;
- that gateway nodes use legacy host routing, and with Globalnet have the identity mark off;
- that Cilium IPsec is not combined with the libreswan cable.

The effective device list and host routing mode are read from the agent, not from `cilium-config`.

### Security model

No Cilium control plane spans clusters, and no Cilium trust relationship is created. That answers the
concern raised on #4106 that a ClusterMesh relationship is bidirectional. Remote clusters stay `world` to
Cilium policy. The handler's only write is one ConfigMap under a name-scoped role. That grant is bound
cluster-wide, so it also covers a `kube-system/ip-masq-agent` ConfigMap on clusters that do not run Cilium,
such as GKE's or the kubernetes-sigs agent's; the handler is active only where Cilium is detected, and
it writes only when `cilium-config` shows the three settings, so nothing writes it, but the permission
exists. The route-agent ServiceAccount is shared by all nodes, but route-agent already
runs with host networking and `NET_ADMIN`, which is the same trust level.

### Coupling to Cilium

The design depends on four pieces of Cilium configuration: `enable-ip-masq-agent` and its file format,
`devices`, `enable-host-legacy-routing`, and `CiliumNodeConfig` (`cilium.io/v2`). No BPF map layout, kvstore
format or identity number is used, so a Cilium upgrade needs no Submariner change unless one of those
surfaces changes.

### Alternatives Considered

**ClusterMesh-shaped ipcache publisher (#4106).** route-agent would run an embedded etcd or kine on every
node and publish remote CIDRs in the ClusterMesh kvstore format, so that Cilium tunnels worker traffic to
the gateway itself. It avoids masquerading in tunnel mode, handles gateway-node pods and gives remote
clusters an identity for policy. But it adds etcd or kine and gRPC to route-agent, an embedded binary in the
tens of MiB. It needs a CA, a cluster id and name and a reserved peer, it conflicts with existing
ClusterMesh and KVStoreMesh setups, and it does nothing for native routing, discovery or Services. It is
kept as a possible later step for identity.

**Direct BPF map programming.** route-agent would write remote entries into Cilium's ipcache map. The map
is recreated on every agent restart without foreign entries (`pkg/ipcache/cell/cell.go:107-129`), and its
layout changes between versions. sanek9's labs for #4106 reached the same conclusion.

**A `CiliumNode` object for the gateway, and other Cilium interfaces.** I did not find a Cilium API meant for
routing a prefix to a node. The ipcache REST endpoint is read-only, CIDR policies give a prefix no tunnel
endpoint, Egress Gateway SNATs, and the VTEP integration is static agent configuration. The closest thing is
a `CiliumNode` that is not backed by a node and carries the gateway's address with the remote CIDRs as its
pod CIDRs. On kind, in tunnel mode, workers then tunnel the traffic to the gateway over Cilium's own VXLAN,
the pod IP is kept and nothing goes through `vx-submariner`, so neither the ip-masq-agent list nor the
device setting is needed. It has four limits. Pods on gateway nodes still need legacy host routing. In
native routing mode the object made no difference. Cilium replaces Submariner's kernel route to the remote
CIDRs with its own, and deleting the object left the worker with no route. And the Cilium operator
garbage-collects such an object after a few minutes unless it carries `cilium.io/do-not-gc`, which my runs
were too short to confirm. The object poses as a node, and the only place I found Cilium doing that is its
own Egress Gateway scale test, so this proposal stays on configuration that Cilium documents.

**Legacy host routing cluster-wide.** This makes the generic path work at a per-packet cost on every
node, but it does not stop masquerading on `vx-submariner`. It is kept as the documented fallback, and per
node as the gateway-node prerequisite.

**Documentation only.** A deployment page could describe the ip-masq-agent list, the pinned devices and the
gateway-node routing, and `subctl diagnose` could check them. That covers everything in this proposal except
the remote ranges, which change at runtime and would have to be edited by hand on every Cilium cluster after
every join or leave. The handler exists for that one list; the rest of the proposal is documentation and
checks already.

| | This proposal | ClusterMesh publisher | Direct BPF | Legacy routing |
|---|---|---|---|---|
| User setup | ip-masq-agent list or pinned devices; gateway node config | cluster id/name, peer Secret | none | one Helm value |
| Code to maintain | ConfigMap reconcile; later discoverer, diagnosis | etcd/kine, TLS, kvstore schema | per-version struct layout | none |
| Dependencies and footprint | none | etcd or kine, gRPC; tens of MiB | none | stack cost on every node |
| Cilium coupling | four config keys | kvstore contract, identity ranges | BPF ABI | one config key |
| Routing modes | both, tested on kind | tunnel only | both | both |
| Remote identity in policy | `world` | cluster-scoped | arbitrary | `world` |

### Backward Compatibility

The handler is additive. On a cluster where it is not active, the plugin name and route-agent's handler set
are the same as today. A Cilium cluster that already works with legacy host routing on every node and
`generic` discovery keeps working.

Discovery is where compatibility needs care, because it changes the plugin name. An older route-agent image
does not know `cilium`, so an operator that reports it leaves that node without the kube-proxy handler.
Detection therefore must not take effect on a cluster whose route-agent predates the name, and how that is
guarded is part of the discovery work.

## Test Results

All runs were on kind. The [appendix](https://gist.github.com/kurtyoon/366ac0e42c60dfbd6b605fa030ffa382) has the matrix
and the source references.

**Without the handler.** Cilium 1.20.2 with Submariner 0.24.2, and later Cilium 1.20.4 with Submariner
0.25.0, in tunnel mode with Kubernetes IPAM, on libreswan and WireGuard, plus a pair with overlapping CIDRs
for Globalnet. Unmodified, pod traffic left with the 240.x address. A pinned device set fixed it, and so did
ip-masq-agent with an explicit list that named the remote CIDRs by hand. ip-masq-agent without a ConfigMap
broke pod connections to the broker. Legacy routing on gateway nodes fixed the pods probed there. A failover
between two gateway nodes by label change lost about 4 s of pod to pod traffic, on the pair without
Globalnet. Headless exports, `GlobalEgressIP` and the reverse direction passed on both cables, and ClusterIP
exports failed. Against a Calico cluster, pod traffic and headless exports passed both ways, and ClusterIP
exports passed into the Calico side, with Calico's node address detection kept off `vx-submariner`
(submariner#4249).

**The four supported configurations.** Cilium 1.20.2, Submariner 0.25.0, libreswan, without Globalnet. The
check is pod to pod traffic over eight paths: client and server pods on gateway and non-gateway nodes, in
both directions. In every row, the two paths between non-gateway nodes pass with the list alone, and all
eight pass once the gateway nodes use legacy host routing. The handler added the remote CIDRs in tunnel mode
with Kubernetes IPAM and in native routing with cluster-pool IPAM; in the other two rows they were in the
list from install time and no handler ran. With the handler, the same eight paths also pass by the
clusterset name of a headless export.

**Handler with Globalnet.** Overlapping CIDRs and libreswan, with `240.0.0.0/8` in the list in place of
`240.0.0.0/4` so that the global CIDRs were left to the handler. It added the other cluster's global CIDR on
both sides. With legacy host routing only, 7 of 24 probe pods on the gateway nodes had an identity with both
mark bits set, and none of the 7 reached its own API server. With the identity mark off all 24 did, and the
eight headless paths passed, with servers seeing the client cluster's global egress IPs.

**VXLAN cable.** With the handler, in tunnel mode with Kubernetes IPAM, with and without Globalnet. The eight
pod paths and the headless exports passed. In the Globalnet run, a host-network pod on each gateway node
reached its API server with the identity mark on and off.

**Handler lifecycle.** In tunnel mode: creating `submariner-global` after the deployment, a chart upgrade of
Cilium with unchanged values, and an uninstall of the other cluster while the local route-agent was held
down, after which the resync removed the stale CIDRs. `subctl uninstall` removed the handler's entries and
annotations and left the user's entries.

**Not run.** The full `verify --only connectivity` suite, network policy, the pod-selector `GlobalEgressIP`,
gateway failover with Globalnet, endpoint routes, and pods behind Cilium's L7 or DNS proxy.

## Implementation Plan

The work starts small, with two phases agreed in review. Phase 1 adds the handler behind a ConfigMap field
and the docs page, with as little change to other components as possible. Phase 2 is the CI gate. During
both, a Cilium cluster is discovered as `generic`, and the user sets the field, passes `--clustercidr` at
join on cluster-pool IPAM and names a Cilium namespace other than `kube-system` in a second key.

The steps after that are decided once the datapath is stable in CI and there is feedback from users, for
example on how much the ClusterIP limitation matters in practice and on which Cilium configurations they
run.

There are no new Go dependencies. The project would maintain one e2e job and one docs page.

### Phase 1: Cilium handler

**Files:** `submariner/pkg/routeagent_driver/handlers/cilium/`, `pkg/routeagent_driver/main.go`;
`submariner-operator/config/rbac/submariner-route-agent/`; the Helm charts; submariner-website
(`operations/deployment/cilium.md`).

The handler described under Design. It is loaded when `enable-cilium-support` is `true` in the
`submariner-global` ConfigMap, the way submariner#3577 added `enable-snat-handler`, and it registers for
`generic`. The user creates or edits that ConfigMap, and route-agent already restarts when it changes. An
optional `cilium-namespace` key names the namespace Cilium runs in and defaults to `kube-system`. On a
cluster that does not run Cilium the field has no effect: the handler finds no `cilium-config` and stays
idle.

Unit tests cover a missing ConfigMap, one without a list, one with user entries, annotation drift, a replaced
config, conflict retry and uninstall. The charts need no template change; their RBAC is regenerated from the
operator at chart release. A first implementation is on two branches:
[submariner](https://github.com/kurtyoon/submariner/tree/4247-cilium-ipmasq-handler) and
[submariner-operator](https://github.com/kurtyoon/submariner-operator/tree/cilium-ip-masq-agent-rbac).

The docs page covers the prerequisites with the values above, the handler's limitations, the policy notes,
and the ClusterIP limitation with its workaround.

### Phase 2: CI gate

**Files:** shipyard (a kind Cilium deploy option), the `submariner` e2e workflow.

Shipyard deploys kind clusters with Cilium through a `cni: cilium` branch that leaves the defaults and the
OVN path unchanged, and sets `enable-cilium-support` in `submariner-global`. One e2e job on `submariner`
runs Cilium ↔ Cilium in tunnel mode, without Globalnet, on every commit. The pod-to-service and
ClusterIP-export specs are skipped toward a Cilium cluster. The job tells the suite which clusters run
Cilium, because discovery reports them as `generic`.

### Later steps

Not scheduled. The candidates are the detection described under Discovery, in
`submariner-operator/pkg/discovery/network/cilium.go`, the post-deployment diagnosis, in `subctl diagnose
cni` or in submariner-diagnostics, a CI job with Globalnet that has the headless spec in
`basic-connectivity`, and Cilium ↔ Calico once shipyard can mix CNIs in one job.

## Done When

Phase 1:

```bash
kubectl -n submariner-operator get configmap submariner-global -o jsonpath='{.data.enable-cilium-support}'
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg bpf ipmasq list
subctl verify --context cluster1 --tocontext cluster2 --only connectivity
```

- [ ] With the field set, the remote CIDRs appear in the ip-masq-agent list and in BPF, and the pod-to-pod
  specs of `verify --only connectivity` pass Cilium ↔ Cilium on kind, in tunnel mode and with native routing.
  Legacy routing is used on gateway nodes only.
- [ ] With the field unset, route-agent's handler set is the same as today.
- [ ] The deployment docs are published.

Phase 2:

- [ ] A Cilium CI job runs on `submariner`: Cilium ↔ Cilium, tunnel mode, without Globalnet.

## Follow-ups

A second proposal, Globalnet service ingress without externalIPs, which I will submit after this one, moves
ClusterIP exports on Globalnet clusters into Submariner's own rules on the exporting gateway, the way
headless exports are handled today, and turns on by itself on Cilium clusters with kube-proxy replacement.
Clusters without Globalnet would need a small ingress range advertised to peers. In my kind runs a
release-0.25 peer did not pick up a subnet added to an existing Endpoint, and rejected an Endpoint that
arrived before its Cluster without re-checking it; I will file both as issues, and that part waits for those
fixes. A later step could give remote clusters a cluster-scoped Cilium identity, so that policy can select
them and Egress Gateway skips them; #4106 is the working reference for that.

Globalnet could stop depending on bits that are already set in the packet mark, for example by matching the
remote destinations directly. That would remove the identity-mark prerequisite, and it is a Globalnet change
outside this proposal. The same collision was reported on
[submariner#3159](https://github.com/submariner-io/submariner/issues/3159#issuecomment-2407651443).
