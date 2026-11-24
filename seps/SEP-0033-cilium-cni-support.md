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

The proposal is smaller than the two approaches discussed on #4106, and it starts with two phases agreed
in review. Phase 1 adds a route-agent handler that keeps remote cluster CIDRs in Cilium's
ip-masq-agent list. The handler is loaded only when a field in the `submariner-global` ConfigMap is set, and
no existing handler changes. Phase 2 adds a Cilium deployment to Shipyard and a CI job. The target design
also has the operator's network discovery detect Cilium, which replaces the manual field; when that is
implemented is decided after the first two phases. The proposal also documents a limitation for exported ClusterIP services on Cilium
clusters. That limitation needs a change outside the CNI integration, which I will propose separately for
Globalnet clusters.

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
Those ranges change whenever a cluster joins or leaves the clusterset, with Globalnet they are allocated at
join time, and the setting lives on every Cilium cluster in the set, so each change is an edit on each of
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

This section says which Cilium configurations and which Submariner connectivity types the proposal supports,
and what the user has to configure. Each status is one of Supported, Not Supported and Not Tested. Supported
means it was run on kind and passed. Not Tested means it is expected to work but has no run behind it.

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

The four supported rows were run with the libreswan cable. WireGuard is Supported in tunnel mode with
Kubernetes IPAM and Not Tested in the other three rows. The VXLAN cable is Not Tested.

- **IPv6:** the handler writes IPv4 CIDRs and requires `enable-ipv4-masquerade`. IPv6 BPF masquerading is beta
  in Cilium, and IPv6 is left for later.
- **kube-proxy with iptables masquerading:** such a cluster keeps today's generic path, and the handler, if
  enabled, stays idle there, because Cilium's ip-masq-agent exists only in the BPF implementation. The
  documentation points it to the existing options: an `ipv4-native-routing-cidr` wide enough to cover the
  remote ranges, `egress-masquerade-interfaces` limited to the uplink, which helps with the WireGuard and VXLAN
  cables but not with libreswan, where traffic to the cable leaves through the uplink, or masquerading turned
  off.

### Supported connectivity

Which Submariner connectivity types work on a Cilium cluster in a supported configuration.

| Connectivity | Without Globalnet | With Globalnet |
|---|---|---|
| **Pod to ClusterIP service export, backends on a Cilium cluster** | **Not Supported** | **Not Supported** |
| Pod to headless service export | Supported | Supported |
| Pod to ClusterIP service export, backends on a cluster with kube-proxy | Supported | Not Tested |
| Service discovery with `clusterset.local` names | Supported | Supported |

- **Pod IP:** without Globalnet, a pod also reaches a remote pod by its IP, which is Supported. A Globalnet
  cluster advertises only its global CIDR, so there a remote pod is reached through a global IP.
- **ClusterIP exports:** an exported ClusterIP service whose backends run on a Cilium cluster is not reachable
  from another cluster, with or without Globalnet. The limitation depends on the cluster that hosts the
  backends, not on the client: a client on a Cilium cluster reaches a ClusterIP export whose backends run on a
  cluster with kube-proxy. Service discovery still answers with the ClusterIP for an export hosted on Cilium,
  so such a service has to be exported headless. With kube-proxy replacement, a request that arrives through
  the cable is not translated on the gateway, so it does not reach a backend. Headless exports are not
  affected and are the workaround. For Globalnet clusters, a follow-up proposal removes the limitation by
  translating ClusterIP exports in Submariner's own rules on the exporting gateway (see Follow-ups).
- **Source address:** without Globalnet the server sees the client pod IP. With Globalnet it sees the client
  cluster's egress global IP.
- **Pods on a gateway node, with Globalnet:** Supported with the identity-mark setting under Prerequisites.
  Pods there whose traffic goes through Cilium's L7 or DNS proxy, and clusters with endpoint routes enabled,
  are Not Tested.
- **Gateway failover:** Supported without Globalnet, where a label change between two gateway nodes lost about
  4 s of pod to pod traffic. Not Tested with Globalnet.
- **Network policy:** Not Tested. Cross-cluster traffic has the `world` identity, so only CIDR rules can match
  it.

### Prerequisites

These are the settings the user makes on a Cilium cluster. The Datapath section says why each is needed.

- **ip-masq-agent with an explicit list.** `ipMasqAgent.enabled=true` and
  `ipMasqAgent.config.nonMasqueradeCIDRs` holding the cluster CIDRs known at install time, plus `240.0.0.0/4`
  while `vx-submariner` is a Cilium device. The handler adds the remote CIDRs later. The default Globalnet
  range `242.0.0.0/8` is inside `240.0.0.0/4`, so with the default range the handler has nothing to add.
  Destinations outside the list are masqueraded as before.
- **Device pinning.** A pinned device set such as `devices: "eth+"`, so that no Submariner device runs Cilium's
  programs. It can replace the list while the setting stays pinned, and it is worth doing alongside the list.
- **Legacy host routing on gateway nodes.** A `CiliumNodeConfig` with `enable-host-legacy-routing: "true"` and
  `nodeSelector: submariner.io/gateway=true`. The setting applies when the Cilium agent starts, so every node
  that can become a gateway must be labelled before Cilium starts there, and labelling a node later needs an
  agent restart.
- **Identity mark off on gateway nodes, with Globalnet.** `enable-identity-mark: "false"` in the same
  `CiliumNodeConfig`. Without it, some pods on a gateway node lose all their egress. A cluster without
  Globalnet does not need it.
- **Cilium support turned on in Submariner.** `enable-cilium-support: "true"` in the `submariner-global`
  ConfigMap. A cluster with cluster-pool IPAM also passes its pod CIDR with `--clustercidr` at join, and a
  Cilium namespace other than `kube-system` goes in the `cilium-namespace` key.
- **Policy for cross-cluster traffic.** A cluster with a default-deny `CiliumNetworkPolicy` needs `world` or
  CIDR rules for it. A `CiliumEgressGatewayPolicy` that covers a remote CIDR must list it in `excludedCIDRs`.
  With the Cilium host firewall, node policy must allow UDP 4800 between nodes and the cable ports from remote
  gateways.

One incompatibility is documented rather than worked around. With Cilium's own IPsec encryption enabled,
Cilium drops raw ESP from sources that are not Cilium nodes, which includes a remote Submariner gateway.
libreswan then needs forced UDP encapsulation (`--force-udp-encaps`, not verified), or another cable
driver.

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

Every NAT in the design is netfilter on the active gateway. Both directions of a flow pass through that
gateway's stack, because packets reach it from a device rather than from a local pod. The exception is a
pod running on the gateway node itself. Its egress is redirected in BPF, which is why gateway nodes need
legacy host routing. With Globalnet, the gateway's SNAT and DNAT sit in the same place. The
[appendix](https://gist.github.com/kurtyoon/366ac0e42c60dfbd6b605fa030ffa382) has hop tables for Globalnet
egress and headless ingress.

Each prerequisite follows from this path.

Remote CIDRs must not be masqueraded on any device that carries them. This is the one hard requirement on
the worker side, and there are two ways to meet it. The recommended way is ip-masq-agent with an explicit
list, which the handler keeps current. The other way is a pinned device set, so that no Submariner device
runs Cilium's programs. This only holds while the setting stays pinned, because Cilium attaches to devices at
runtime and `vx-submariner` is created after the agent starts. Pinning is still worth doing alongside the
list. Otherwise Cilium takes `vx-submariner`'s address as a NodePort address and applies host-firewall policy
to it.

Legacy host routing is only about pods that run on a gateway node. Traffic that reaches the gateway from
another node already goes through the host stack with no extra setting. A local pod's egress is redirected
in BPF along table 150 and skips the gateway's netfilter hooks. With libreswan this also skips the IPsec
output policy and the packet is dropped. With Globalnet, on any cable, it skips the Globalnet SNAT, and a
backend's reply skips the DNAT reversal. WireGuard without Globalnet happens to work. A `CiliumNodeConfig`
that selects the gateway nodes fixed every gateway-node case on kind, including Submariner's own pods.
Keeping workloads off gateway nodes with a taint is an untested alternative.

Globalnet adds one more setting on gateway nodes. When Cilium passes pod traffic to the stack, it writes the
pod's identity into the upper 16 bits of the packet mark (`bpf/bpf_lxc.c:1204`). Globalnet's egress rules
match `0xC0000` in the same mark to find traffic bound for a remote cluster, and they do not clear those bits
first. With legacy host routing, a gateway-node pod whose identity has both bits set therefore has every new
connection SNATed to the cluster's global egress IP, including connections inside its own cluster. On kind
that was 7 of 24 pods. With `enable-identity-mark: "false"` Cilium does not write the identity on that path
and looks it up by source address instead. Traffic that Cilium's L7 or DNS proxy forwards still carries the
identity in the mark.

### Discovery (submariner-operator)

For now a Cilium cluster is discovered as `generic`, and Cilium support is turned on with
`enable-cilium-support` in the `submariner-global` ConfigMap. The target design adds Cilium detection to the
operator's network discovery, including the cluster's default pod and service CIDRs, so that the user does
not have to set the field, pass `--clustercidr` at join or name the Cilium namespace.

A new `pkg/discovery/network/cilium.go` is appended after the KindNet discoverer, so it runs only when
OpenShift4, OVN-Kubernetes, Weave, Canal, Calico, Flannel and KindNet did not match, and before the generic
fall-through. A Calico cluster that also carries a `cilium` DaemonSet, during a migration or in chaining,
therefore stays Calico and keeps its IPPool handler. NotFound and Forbidden on the discoverer's lookups mean
no match, not an error, because a discoverer error aborts the reconcile.

It detects Cilium only when all of the following hold. Otherwise the cluster stays `generic` with Cilium
support off, and a log line names the step:

1. Exactly one DaemonSet whose pod template carries `k8s-app=cilium` has ready pods. None or more than one
   means not detected. The namespace comes from that DaemonSet, which covers a non-default
   namespace and a leftover `cilium-config` without an agent.
2. The cluster is not a managed Cilium distribution: no `anetd` DaemonSet (GKE Dataplane V2) and no
   `ipam: delegated-plugin` or AKS pod labels (Azure CNI Powered by Cilium). Both keep their own masquerade
   configuration, which the handler must not rewrite. Their markers come from the published manifests and
   are not verified on a live cluster.
3. `cilium-config` exists in that namespace, `cni-chaining-mode` is absent, `none` or `portmap`, and
   `cni-chaining-target` is not set. Any other chaining mode means another CNI owns pod networking.
4. The agent's `desiredNumberScheduled` covers every Linux node; otherwise not detected, with a warning.

The service CIDR keeps coming from the generic sources, which do not depend on the CNI. Only after the
steps above, when IPAM is `cluster-pool` and the generic sources are empty, it reads the pod CIDR from
`cluster-pool-ipv4-cidr` and `cluster-pool-ipv6-cidr`, splitting on spaces and commas. In chaining mode
`ipam.mode` can stay at its `cluster-pool` default while another CNI allocates the addresses, so the chart
still renders `cluster-pool-ipv4-cidr` as `10.0.0.0/8`; reading it before the chaining check would advertise
that unused range. It also keeps `routing-mode`, `kube-proxy-replacement`, `enable-bpf-masquerade`,
`enable-host-legacy-routing`, `enable-ip-masq-agent` and `devices` in the operator's `ClusterNetwork`, where
the follow-up proposal's default comes from; a prerequisite check does not read them, it asks the agent.

Cilium falls back to legacy host routing, with only an info log, when kube-proxy replacement is off or
masquerading runs in iptables rather than BPF. A check therefore reads the effective mode from the agent
rather than from the ConfigMap.

How the operator hands the result to route-agent is decided when discovery is implemented. The simplest
form keeps the plugin name `generic` and has the operator write the two keys the handler already reads.
Reporting `cilium` as the plugin name would be more visible in `subctl show networks`, but it needs three
more changes: `cilium` in the kube-proxy handler's plugin list, a route-agent fallback for plugin names it
does not know, and an operator check for older route-agent images.

OpenShift with Cilium is out of scope. The OpenShift4 discoverer copies `spec.networkType`, so such a
cluster reports `Cilium` today, a name the kube-proxy handler does not register for, which leaves it without
a Submariner datapath. This proposal does not change that.

### route-agent Cilium handler

A new handler in `pkg/routeagent_driver/handlers/cilium` is appended in route-agent's `main.go` only when
`enable-cilium-support` is `true` in the `submariner-global` ConfigMap, the way submariner#3577 added
`enable-snat-handler`. That ConfigMap is optional and nothing creates it today, so the user creates or edits
it. route-agent already restarts when it changes: on kind, creating it after the deployment
restarted the route-agent pods within a few seconds, with no manual step. A second, optional key
`cilium-namespace` names the namespace Cilium runs in and defaults to `kube-system`.

The handler registers for `generic`, which is what a Cilium cluster is discovered as. It is additive, like
the Calico IPPool handler. No existing handler and no plugin list changes, and with the field unset
route-agent behaves as it does today.

Its job is to keep every remote cluster CIDR in Cilium's ip-masq-agent list. With ip-masq-agent enabled
and no ConfigMap, Cilium uses a built-in list that covers all of RFC 1918 and the reserved ranges
(`pkg/ipmasq/ipmasq.go:26-45`). That stops masquerading toward every private address outside the
cluster, not only toward remote clusters. On kind it broke pod connections to the broker API server: the
replies to the pod IP followed Submariner's route and were dropped on the far gateway. An explicit list
avoids that, so the proposal supports only an explicit list.

The desired set is the IPv4 subnets of every remote `Endpoint`, recomputed from all of them each time, so a
remote gateway failover never drops a CIDR. If the `ip-masq-agent` ConfigMap does not exist or has no list,
the built-in list is in effect, so the handler logs a warning and writes nothing. It does not create the
ConfigMap itself, because a `create` permission cannot be limited by resource name.

When the list exists, the handler appends the desired CIDRs that are missing after the user's entries, and
writes only when the list changes. It records what it added in the annotation
`submariner.io/non-masquerade-cidrs`, so user entries are never touched. Entries it added and no longer
wants are removed on a remote Endpoint update or removal and on a five minute resync, not on every event.
After a restart the remote Endpoints arrive one at a time, and removing then would briefly drop subnets that
are still in use. The resync also covers Cilium settings that are turned on after route-agent starts.

A second annotation, `submariner.io/non-masquerade-config-checksum`, holds a checksum of the config the
handler wrote. When something else replaces the config, for example a chart upgrade that renders the list
again, the checksum no longer matches and the handler stops treating the entries in the first annotation as
its own, since whatever is in the new list was put there by its new writer. The handler also watches the
ConfigMap, so the remote CIDRs come back without waiting for an endpoint event. Cilium reloads the file on
change, and propagation is the kubelet ConfigMap sync, about a minute.

Only the active gateway writes, with conflict retries. A gateway that steps down removes nothing, since
removing entries would open a masquerade window. Uninstall removes only the annotated entries and the two
annotations.

The handler does no I/O in its constructor. `Init` starts the ConfigMap watch and does not fail when the
ConfigMap or the permission is missing, because an `Init` error is fatal for route-agent on every node. It
writes only when `cilium-config` has `enable-ip-masq-agent`, `enable-bpf-masquerade` and
`enable-ipv4-masquerade` all on; otherwise it logs the setting and stays idle. `Uninstall` runs on every node
without gateway state, so it is idempotent and retries on conflict.

The route-agent ClusterRole already has `get` and `list` on all ConfigMaps. The new rule adds `watch` and
`update` on `configmaps` with `resourceNames: [ip-masq-agent]`; a name-restricted watch is authorized only
with the field selector `metadata.name=ip-masq-agent`, so the informer uses it. A Role in the Cilium
namespace would have no creator on the Helm and OLM paths. The handler does not touch ipcache,
`cilium-config`, `CiliumNode` or any BPF map, and it uses the Kubernetes client only, with no Cilium client
library. `ipv4-native-routing-cidr` is not an alternative to the handler, since it is a single static prefix
and remote CIDRs change at runtime.

Known limitations of the handler:

- The list is one string field, `data.config`, with two writers. Helm 4 applies charts server-side, and on
  kind a `helm upgrade` of Cilium with unchanged values failed with a field conflict on that field between
  the managers `helm` and `submariner-route-agent`. It went through with `--force-conflicts`, after which the
  handler added the remote CIDRs back in the next ConfigMap version and traffic kept passing. I expect any
  tool that applies the Cilium chart server-side to hit the same; Helm 3, Argo CD and Flux were not run.
  Putting the remote CIDRs in the Helm values avoids the conflict, because the handler then has nothing to
  write. The docs describe both.
- If `enable-cilium-support` is turned off before uninstall, the entries stay, because the handler is no
  longer loaded to remove them. The docs say to uninstall first or to remove the entries by hand.
- If the ClusterRole rule is missing, the handler only logs the error.

### Globalnet

Globalnet needs no change of its own. Egress through the cluster and namespace `GlobalEgressIP`s,
headless-service ingress and the gateway health check are netfilter on the gateway, and both directions pass
through it. The pod-selector `GlobalEgressIP` uses the same path.

For ClusterIP exports, the Cilium-side options that were tried either hide the client's global IP from
backends or need a hand-tuned MSS clamp. They are not part of this proposal; the follow-up proposal moves the
translation into Submariner's own gateway rules instead.

### Policy and identity

Cross-cluster traffic keeps the `world` identity, including Globalnet-translated traffic. No Cilium identity
is assigned to a remote cluster, so identity-based policy cannot select it. Giving remote clusters a Cilium
identity is what the ClusterMesh-shaped publisher would add; see Follow-ups.

### Prerequisite checks

Once discovery detects Cilium, the prerequisites can be checked for the user. `subctl diagnose cni` is where
CNI validation lives, and it can cover Cilium the way it covers OVN-Kubernetes. The checks could also live in
[submariner-diagnostics](https://github.com/submariner-io/submariner-diagnostics), which needs no `subctl`
change. Which of the two is decided when the checks are written. `subctl join` gets no Cilium check. In
either place the checks are:

- that ip-masq-agent is enabled with a ConfigMap and every remote CIDR is in the effective list, or that the
  device set is pinned;
- that gateway nodes use legacy host routing;
- that Cilium IPsec is not combined with the libreswan cable.

The effective device list and host routing mode are read from the agent, not from `cilium-config`.

### Security model

No Cilium control plane spans clusters, and no Cilium trust relationship is created. That answers the
concern raised on #4106 that a ClusterMesh relationship is bidirectional. Remote clusters stay `world` to
Cilium policy. The handler's only write is one ConfigMap under a name-scoped role. That grant is bound
cluster-wide, so it also covers a `kube-system/ip-masq-agent` ConfigMap on clusters that do not run Cilium,
such as GKE's or the kubernetes-sigs agent's; the handler is not loaded there unless the field is set, and
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
| Code to maintain | ConfigMap reconcile; later discoverer, checks | etcd/kine, TLS, kvstore schema | per-version struct layout | none |
| Dependencies and footprint | none | etcd or kine, gRPC; tens of MiB | none | stack cost on every node |
| Cilium coupling | four config keys | kvstore contract, identity ranges | BPF ABI | one config key |
| Routing modes | both, tested on kind | tunnel only | both | both |
| Remote identity in policy | `world` | cluster-scoped | arbitrary | `world` |

### Backward Compatibility

Nothing changes unless `enable-cilium-support` is set: the handler is not loaded, the plugin name
stays `generic`, and every existing handler runs as before. With the field set on a cluster that does not
run Cilium, the handler finds no `cilium-config` and stays idle. A Cilium cluster that already runs with
legacy host routing and `generic` discovery keeps working with or without the field.

Discovery is where compatibility needs care, and only if it changes the plugin name. An operator that sends
`cilium` to an older route-agent image would lose the kube-proxy handler there, which is why the Discovery
section calls keeping `generic` the simplest form.

## Implementation Plan

The work starts small, with two phases agreed in review. Phase 1 adds the handler behind the ConfigMap field
and documents the prerequisites, with as little change to other components as possible. Phase 2 adds a
Cilium deployment to Shipyard and a CI job. During both, a Cilium cluster is discovered as `generic`, and the
user sets the field, passes `--clustercidr` at join on cluster-pool IPAM and names a Cilium namespace other
than `kube-system` in the second key.

The steps after that are decided once the datapath is stable in CI and there is feedback from users, for
example on how much the ClusterIP limitation matters in practice and on which Cilium configurations they
run. Discovery, which removes the three manual steps, and the prerequisite checks are the candidates.

There are no new Go dependencies and `go.mod` is unchanged. The handler is one reconcile loop. The project
would maintain one e2e job with two matrix entries, the two latest Cilium minors, and one docs page.

### Lab results

The first kind lab used Cilium 1.20.2 and Submariner 0.24.2 with libreswan and WireGuard, plus a second pair
of clusters with overlapping CIDRs for Globalnet. Unmodified, pod traffic left with the 240.x address. A
pinned device set fixed it, and so did ip-masq-agent. The built-in list broke pod-to-broker traffic, which
a later run with an explicit list did not. Legacy routing on gateway nodes fixed pods there. Against a
Calico cluster, pod traffic and headless exports passed both ways, and ClusterIP exports passed into the
Calico side. That needed Calico's node address detection kept off `vx-submariner`, which is not specific
to Cilium: [calico#14300](https://github.com/projectcalico/calico/pull/14300) adds the device to Calico's
default exclude list, and [submariner#4249](https://github.com/submariner-io/submariner/issues/4249) tracks
the Submariner side. A failover between two gateway nodes by label change lost about 4 s of pod-to-pod
traffic, on the pair without Globalnet. A later run on Submariner 0.25.0 and Cilium 1.20.4 with the same
prerequisites gave the same results: the explicit list was in effect in BPF, headless exports,
`GlobalEgressIP` and the reverse direction passed on both cables, and ClusterIP exports still failed. The
[appendix](https://gist.github.com/kurtyoon/366ac0e42c60dfbd6b605fa030ffa382) has the matrix and the source
references.

The four rows of the Supported configurations table were then run on kind with Cilium 1.20.2, Submariner
0.25.0 and libreswan, without Globalnet. The Phase 1 handler added the remote CIDRs in tunnel mode with
Kubernetes IPAM and in native routing with cluster-pool IPAM; in the other two rows they were in the list
from install time and no handler ran. The check is pod to pod traffic over eight paths: client and server
pods on gateway and non-gateway nodes, in both directions. In every row, two of the eight pass with the list
alone, the ones between non-gateway nodes, and all eight pass once the gateway nodes use legacy host routing.
With the handler, the same eight paths also pass by the clusterset name of a headless export. Earlier runs
with Globalnet and with WireGuard had the remote CIDRs in the list by hand; on the Globalnet pair, clients
and backends on both node types worked on both cables, servers saw the client cluster's egress IPs, and the
pod-selector `GlobalEgressIP` was not run.

The handler was then run with Globalnet, on a pair with overlapping CIDRs and libreswan, with `240.0.0.0/8`
in the list in place of `240.0.0.0/4` so that the global CIDRs were left to the handler. It added the other
cluster's global CIDR on both sides. With legacy host routing only, 7 of 24 probe pods on the gateway nodes
had an identity with both mark bits set. None of the 7 reached its own API server and the other 17 did; an
earlier run had lost one cluster's operator and lighthouse agent the same way. With
`enable-identity-mark: "false"` on the gateway nodes all 24 did, and the eight headless paths passed, with
servers seeing the client cluster's global egress IPs. Pods behind Cilium's L7 or DNS proxy and endpoint
routes were not run.

Besides the eight paths, three cases were run in tunnel mode with the handler: creating `submariner-global`
after the deployment, a chart upgrade of Cilium with unchanged values,
and an uninstall of the other cluster while the local route-agent was held down, after which the resync
removed the stale CIDRs about five minutes after route-agent came back. `subctl uninstall` removed the
handler's entries and annotations and left the user's entries. Still to run are the full
`verify --only connectivity` suite, network policy and the VXLAN cable.

### Phase 1: Cilium handler

**Files:** `submariner/pkg/routeagent_driver/handlers/cilium/`, `pkg/routeagent_driver/main.go`;
`submariner-operator/config/rbac/submariner-route-agent/`; the Helm charts.

The reconcile with annotation ownership, the checksum, the resync and a ConfigMap watch, loaded behind
`enable-cilium-support`. Unit tests cover a missing ConfigMap, one without a list, one with user entries,
annotation drift, a replaced config, conflict retry and uninstall. The charts need no template change; their
RBAC is regenerated from the operator at chart release. A first implementation is on two branches:
[submariner](https://github.com/kurtyoon/submariner/tree/4247-cilium-ipmasq-handler) and
[submariner-operator](https://github.com/kurtyoon/submariner-operator/tree/cilium-ip-masq-agent-rbac).

### Phase 2: Shipyard and CI

**Files:** shipyard (a kind Cilium deploy option), `submariner/test/e2e/dataplane`, submariner-website
(`operations/deployment/cilium.md`).

Shipyard deploys kind clusters with Cilium through a `cni: cilium` branch that leaves the defaults and the
OVN path unchanged, and sets `enable-cilium-support` in `submariner-global`. One e2e job on `submariner`
runs Cilium ↔ Cilium, in tunnel mode by default, on every commit. Cilium ↔ Calico follows once shipyard can
mix CNIs in one job. In the dataplane suite, the existing headless Globalnet spec gets the `basic` label,
which adds its two specs to `basic-connectivity` on every Globalnet pair. The pod-to-service and
ClusterIP-export specs are skipped toward a Cilium cluster. The job tells the suite which
clusters run Cilium, because discovery reports them as `generic`. Five of the six Globalnet groups use
GlobalServiceIP, so toward a Cilium target the skip also removes their egress-selector and host-network
cases. On a Globalnet pair the ClusterIP-export case comes back with the follow-up proposal's specs; without
Globalnet it stays skipped, as does the raw pod-to-service spec. The docs cover the field, the
prerequisites, the handler's limitations, the policy notes, and the ClusterIP limitation with its
workaround.

### Later steps

Not scheduled. The candidates are the detection described under Discovery, in
`submariner-operator/pkg/discovery/network/cilium.go`, and the prerequisite checks, in `subctl diagnose cni`
or in submariner-diagnostics. Whether and in what order they are implemented is decided after Phases 1 and 2,
from the CI results and user feedback.

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

Phase 2:

- [ ] A Cilium CI job runs on `submariner`, and the deployment docs are published.
- [ ] On a Globalnet pair, the headless spec passes, and the ClusterIP-export specs are skipped toward
  Cilium.

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
