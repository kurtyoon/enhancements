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

The proposal is smaller than the two approaches discussed on #4106, and it is delivered in three phases
agreed in review. Phase 1 adds a route-agent handler that keeps remote cluster CIDRs in Cilium's
ip-masq-agent list. The handler is loaded only when a field in the `submariner-global` ConfigMap is set, and
no existing handler changes. Phase 2 adds a Cilium deployment to Shipyard and a CI job. Phase 3 adds an
operator discoverer for Cilium, which replaces the manual field, and gives `subctl` a short list of Cilium
settings to check. The proposal also documents a limitation for exported ClusterIP services on Cilium
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
one part a document cannot do. The prerequisites that do not change at runtime stay as documentation and
`subctl` checks.

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

## Design

### Scope

IPv4 only. The Phase 1 handler writes IPv4 CIDRs and requires `enable-ipv4-masquerade`. IPv6 BPF
masquerading is beta in Cilium, and IPv6 is left for later.

Access to an exported ClusterIP service whose backends run on a Cilium cluster is a documented limitation,
with and without Globalnet. With kube-proxy replacement, a request that arrives through the cable is not
translated on the gateway, so it does not reach a backend. The Cilium-side options that were tried either
hide the client's address from backends or need a hand-tuned MSS clamp (see Globalnet below).

Headless exports are not affected and are the workaround. For Globalnet clusters, a follow-up proposal
removes the limitation by translating ClusterIP exports in Submariner's own rules on the exporting gateway
(see Follow-ups). Clusters without Globalnet keep the headless workaround.

Cilium clusters that keep kube-proxy and masquerade with iptables, with BPF masquerading off, are also outside
this proposal. They keep today's generic path, and the handler, if enabled, stays idle there, because Cilium's
ip-masq-agent exists only in the BPF implementation. `subctl diagnose` reports that masquerading is handled
by iptables and points to the existing options: an `ipv4-native-routing-cidr` wide enough to cover the remote
ranges, `egress-masquerade-interfaces` limited to the uplink, which helps with the WireGuard and VXLAN cables
but not with libreswan, where traffic to the cable leaves through the uplink, or masquerading turned off.

### Supported configurations

The target is Cilium 1.16 to 1.20 on IPv4 with kube-proxy replacement, BPF masquerading, ip-masq-agent with
an explicit list, and legacy host routing on the gateway nodes (see Prerequisites). The table shows what was
run on kind with Cilium 1.20.2 and Submariner 0.25.0. "Handler" means the Phase 1 handler added the remote
CIDRs to the list. "By hand" means they were in the list from install time and no handler ran.

| Routing mode | IPAM | Remote CIDRs added | Result |
|---|---|---|---|
| tunnel | Kubernetes host-scope | handler | pass |
| tunnel | cluster-pool | by hand | pass |
| native | Kubernetes host-scope | by hand | pass |
| native | cluster-pool | handler | pass |

The check is pod to pod traffic over eight paths: client and server pods on gateway and non-gateway nodes, in
both directions. In every row, two of the eight pass with the list alone, the ones between non-gateway nodes,
and all eight pass once the gateway nodes use legacy host routing. The server sees the client pod IP.

These runs used libreswan without Globalnet. WireGuard and Globalnet were run earlier in tunnel mode with
Kubernetes IPAM, with the prerequisites set by hand (see Lab results). Not run: the VXLAN cable, and WireGuard
or Globalnet with native routing or cluster-pool IPAM. Out of scope: IPv6, ClusterIP exports into a Cilium
cluster, CNI chaining, the managed Cilium distributions and OpenShift with Cilium.

### Supported connectivity

What a Cilium cluster in the target configuration gets from Submariner. "Yes" was run on kind and passed.
"No" is the documented limitation. "Not run" is expected to work but has no run behind it.

| Capability | Without Globalnet | With Globalnet |
|---|---|---|
| Pod to remote pod IP | yes | not applicable |
| Pod to headless export | yes | yes |
| Pod to ClusterIP export, backends on Cilium | no | no |
| Pod to ClusterIP export, backends on another CNI | yes | not run |
| `clusterset.local` DNS names | yes | yes |
| Cluster and namespace `GlobalEgressIP` | not applicable | yes |
| Pod-selector `GlobalEgressIP` | not applicable | not run |
| Gateway failover | yes | not run |
| Network policy on cross-cluster traffic | not run | not run |

- **Source address:** without Globalnet the server sees the client pod IP. With Globalnet it sees the client
  cluster's egress global IP.
- **ClusterIP exports:** the limitation depends on the cluster that hosts the backends, not on the client. A
  client on a Cilium cluster reaches a ClusterIP export whose backends run on a cluster with kube-proxy. That
  row was run against Calico. Service discovery still answers with the ClusterIP for an export hosted on
  Cilium, so such a service has to be exported headless.
- **Gateway failover:** a label change between two gateway nodes lost about 4 s of pod to pod traffic.
- **Network policy:** cross-cluster traffic has the `world` identity, so only CIDR rules can match it (see
  Policy and identity).
- **What the handler was run with:** pod to remote pod IP, without Globalnet. The other rows were run with
  the remote CIDRs put in the list by hand. With Globalnet the remote Endpoints advertise the global CIDRs,
  so those are what the handler adds; that combination has not been run.

### Datapath

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
pod running on the gateway node itself. Its egress is redirected in BPF, which is handled by a
prerequisite below. With Globalnet, the gateway's SNAT and DNAT sit in the same place. The
[appendix](https://gist.github.com/kurtyoon/366ac0e42c60dfbd6b605fa030ffa382) has hop tables for Globalnet
egress and headless ingress.

### Discovery (submariner-operator, Phase 3)

Discovery is deferred to Phase 3, after the datapath has run in CI. In Phases 1 and 2 a Cilium cluster is
still discovered as `generic`. The handler is turned on by the ConfigMap field, a cluster with cluster-pool
IPAM passes its pod CIDR with `--clustercidr` at join, and a Cilium namespace other than `kube-system` goes
in a second key. Discovery removes those three manual steps. The rules below are the design for it and may
change with what Phases 1 and 2 show.

A new `pkg/discovery/network/cilium.go` is appended after the KindNet discoverer, so it runs only when
OpenShift4, OVN-Kubernetes, Weave, Canal, Calico, Flannel and KindNet did not match, and before the generic
fall-through. A Calico cluster that also carries a `cilium` DaemonSet, during a migration or in chaining,
therefore stays Calico and keeps its IPPool handler. NotFound and Forbidden on the discoverer's lookups mean
no match, not an error, because a discoverer error aborts the reconcile.

It concludes `cilium` only when all of the following hold, and reports `generic` otherwise, with a log line
naming the step:

1. Exactly one DaemonSet whose pod template carries `k8s-app=cilium` has ready pods. None means no match;
   more than one means `generic`. The namespace comes from that DaemonSet, which covers a non-default
   namespace and a leftover `cilium-config` without an agent.
2. The cluster is not a managed Cilium distribution: no `anetd` DaemonSet (GKE Dataplane V2) and no
   `ipam: delegated-plugin` or AKS pod labels (Azure CNI Powered by Cilium). Both keep their own masquerade
   configuration, which the handler must not rewrite. Their markers come from the published manifests and
   are not verified on a live cluster.
3. `cilium-config` exists in that namespace, `cni-chaining-mode` is absent, `none` or `portmap`, and
   `cni-chaining-target` is not set. Any other chaining mode means another CNI owns pod networking.
4. The agent's `desiredNumberScheduled` covers every Linux node; otherwise `generic` with a warning.

Only after that, when IPAM is `cluster-pool` and the generic sources are empty, it reads the pod CIDR from
`cluster-pool-ipv4-cidr` and `cluster-pool-ipv6-cidr`, splitting on spaces and commas. In chaining mode
`ipam.mode` can stay at its `cluster-pool` default while another CNI allocates the addresses, so the chart
still renders `cluster-pool-ipv4-cidr` as `10.0.0.0/8`; reading it before the chaining check would advertise
that unused range. It also keeps `routing-mode`, `kube-proxy-replacement`, `enable-bpf-masquerade`,
`enable-host-legacy-routing`, `enable-ip-masq-agent` and `devices` in the operator's `ClusterNetwork`, where
the follow-up proposal's default comes from; diagnose does not read them, it asks the agent.

Cilium falls back to legacy host routing, with only an info log, when kube-proxy replacement is off or
masquerading runs in iptables rather than BPF. Diagnose therefore reads the effective mode from the agent
rather than from the ConfigMap.

How the operator hands the result to route-agent is left open until then. The simplest form keeps the
plugin name `generic` and has the operator write the two keys the handler already reads. Reporting `cilium`
as the plugin name would be more visible in `subctl show networks`, but it brings back three changes that
the phased plan removed: `cilium` in the kube-proxy handler's plugin list, a route-agent fallback for plugin
names it does not know, and an operator check for older route-agent images.

OpenShift with Cilium is out of scope. The OpenShift4 discoverer copies `spec.networkType`, so such a
cluster reports `Cilium` today, a name the kube-proxy handler does not register for, which leaves it without
a Submariner datapath. This proposal does not change that.

### route-agent Cilium handler (Phase 1)

A new handler in `pkg/routeagent_driver/handlers/cilium` is appended in route-agent's `main.go` only when
`enable-cilium-support` is `true` in the `submariner-global` ConfigMap, the way submariner#3577 added
`enable-snat-handler`. That ConfigMap is optional and nothing creates it today, so in Phase 1 the user
creates or edits it. route-agent already restarts when it changes: on kind, creating it after the deployment
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

### Prerequisites

These are Cilium settings the user makes. Phase 1 documents them and Phase 3 adds the `subctl` checks.

Remote CIDRs must not be masqueraded on any device that carries them. This is the one hard requirement on
the worker side, and there are two ways to meet it. The recommended way is
ip-masq-agent with an explicit list (`ipMasqAgent.enabled=true` and `ipMasqAgent.config.nonMasqueradeCIDRs`).
The list holds the cluster CIDRs known at install time, plus `240.0.0.0/4` while `vx-submariner` is a
Cilium device, and the handler adds remote CIDRs later. Destinations outside the list are masqueraded as
before.

The other way is a pinned device set such as `devices: "eth+"`, so that no Submariner device runs Cilium's
programs. This only holds while the setting stays pinned, because Cilium attaches to devices at runtime
and `vx-submariner` is created after the agent starts. Pinning is still worth doing alongside the
ip-masq-agent list. Otherwise Cilium takes `vx-submariner`'s address as a NodePort address and applies
host-firewall policy to it.

The other prerequisite is only about pods that run on a gateway node. Traffic that reaches the gateway from
another node already goes through the host stack with no extra setting. A local pod's egress is redirected
in BPF along table 150 and skips the gateway's netfilter hooks. With libreswan this also skips the IPsec
output policy and the packet is dropped. With Globalnet, on any cable, it skips the Globalnet SNAT, and a
backend's reply skips the DNAT reversal. WireGuard without Globalnet happens to work.

The fix is legacy host routing on gateway nodes only, through a `CiliumNodeConfig` with
`enable-host-legacy-routing: "true"` and `nodeSelector: submariner.io/gateway=true`. It fixed every
gateway-node case on kind, including Submariner's own pods. The setting applies when the Cilium agent
starts. Every node that can become a gateway must therefore be labelled before Cilium starts there, and
labelling a node later needs an agent restart. Keeping workloads off gateway nodes with a taint is an
untested alternative.

In Phase 3, `subctl join` warns when it cannot confirm the masquerade prerequisite. `subctl diagnose` checks
the masquerade prerequisite, the device set and the gateway-node routing, reading the effective device list
and host routing mode from the agent through pod exec, as the OVN check does for its database version.

One incompatibility is documented rather than worked around. With Cilium's own IPsec encryption enabled,
Cilium drops raw ESP from sources that are not Cilium nodes, which includes a remote Submariner gateway.
libreswan then needs forced UDP encapsulation (`--force-udp-encaps`, not verified), or another cable
driver.

### Globalnet

Egress through the cluster and namespace `GlobalEgressIP`s, headless-service ingress and the gateway
health check work unchanged, because they are netfilter on the gateway and both directions pass through
it. On kind, with overlapping CIDRs, they worked for clients and backends on both node types, with WireGuard
and libreswan, once the prerequisites were in place. Servers saw the client cluster's egress IPs. The
pod-selector `GlobalEgressIP` uses the same path but was not run.

ClusterIP exports do not work, for the reason given in Scope. The Cilium-side options that were tried
either hide the client's global IP from backends or need a hand-tuned MSS clamp. They are not part of this
proposal; the follow-up proposal moves the translation into Submariner's own gateway rules instead.

### Policy and identity

Cross-cluster traffic keeps the `world` identity, including Globalnet-translated traffic. Clusters with a
default-deny `CiliumNetworkPolicy` need `world` or CIDR rules for it. A `CiliumEgressGatewayPolicy` that
covers a remote CIDR must list it in `excludedCIDRs`. With the Cilium host firewall, node policy must allow
UDP 4800 between nodes and the cable ports from remote gateways. All of this is documented, not changed.
Giving remote clusters a Cilium identity is what the ClusterMesh-shaped publisher would add; see
Follow-ups.

### subctl (Phase 3)

When the cluster is discovered as Cilium, `subctl join` warns if neither the ip-masq-agent list nor a pinned
device set is configured. `subctl diagnose cni` accepts either, as join does: it checks that ip-masq-agent
is enabled with a ConfigMap and that every remote CIDR is in the effective list, or that the device set is
pinned. It also checks pods on gateway nodes with BPF host routing, and Cilium IPsec together with
libreswan. If Phase 3 reports `cilium` as the plugin name, it must also be added to the plugin lists in
diagnose (`supportedNetworkPlugins`) and gather (`networkPluginCNIType`); without that, diagnose fails with
"unsupported CNI plugin" and gather drops the iptables and nftables dumps that Cilium clusters get today as
`generic`. The Cilium checks use the pod-exec helper in `internal/pods` and leave the OVN check untouched.

### Security model

No Cilium control plane spans clusters, and no Cilium trust relationship is created. That answers the
concern raised on #4106 that a ClusterMesh relationship is bidirectional. Remote clusters stay `world` to
Cilium policy. The handler's only write is one ConfigMap under a name-scoped role. That grant is bound
cluster-wide, so it also covers a `kube-system/ip-masq-agent` ConfigMap on clusters that do not run Cilium,
such as GKE's or the kubernetes-sigs agent's; the handler is not loaded there unless the field is set, and
it writes only when `cilium-config` shows the three settings, so nothing writes it, but the permission
exists. The route-agent ServiceAccount is shared by all nodes, but route-agent already
runs with host networking and `NET_ADMIN`, which is the same trust level.

### Cilium versions and maintenance

The design depends on four pieces of Cilium configuration: `enable-ip-masq-agent` and its file format,
`devices`, `enable-host-legacy-routing`, and `CiliumNodeConfig` (`cilium.io/v2`, since 1.16). That makes the
supported range Cilium 1.16 to 1.20. No BPF map layout, kvstore format or identity number is
used, so a Cilium upgrade needs no Submariner change unless one of those surfaces changes. Only 1.20 has
been run. The Phase 2 e2e job runs a matrix of the two latest Cilium minors.

There are no new Go dependencies and `go.mod` is unchanged. The handler is one reconcile loop, and the
discoverer and diagnose checks follow the existing Calico and OVN code. The project would maintain one e2e
job with two matrix entries and one docs page.

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
| Code to maintain | ConfigMap reconcile; later discoverer, diagnose | etcd/kine, TLS, kvstore schema | per-version struct layout | none |
| Dependencies and footprint | none | etcd or kine, gRPC; tens of MiB | none | stack cost on every node |
| Cilium coupling | four config keys | kvstore contract, identity ranges | BPF ABI | one config key |
| Routing modes | both, tested on kind | tunnel only | both | both |
| Remote identity in policy | `world` | cluster-scoped | arbitrary | `world` |

### Backward Compatibility

Phase 1 changes nothing unless `enable-cilium-support` is set: the handler is not loaded, the plugin name
stays `generic`, and every existing handler runs as before. With the field set on a cluster that does not
run Cilium, the handler finds no `cilium-config` and stays idle. A Cilium cluster that already runs with
legacy host routing and `generic` discovery keeps working with or without the field.

Phase 3 is where compatibility needs care, and only if it changes the plugin name. An operator that sends
`cilium` to an older route-agent image would lose the kube-proxy handler there, which is why the Discovery
section prefers keeping `generic`.

## Implementation Plan

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

The Phase 1 handler was then run on kind with Cilium 1.20.2 and libreswan, without Globalnet, in the
combinations of the Supported configurations table. Besides the eight paths, three cases were run in tunnel
mode: creating `submariner-global` after the deployment, a chart upgrade of Cilium with unchanged values,
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
ClusterIP-export specs are skipped toward a Cilium cluster. Until Phase 3 the job tells the suite which
clusters run Cilium; after it, the specs read the discovered plugin. Five of the six Globalnet groups use
GlobalServiceIP, so toward a Cilium target the skip also removes their egress-selector and host-network
cases. On a Globalnet pair the ClusterIP-export case comes back with the follow-up proposal's specs; without
Globalnet it stays skipped, as does the raw pod-to-service spec. The docs cover the field, the
prerequisites, the handler's limitations, the policy notes, and the ClusterIP limitation with its
workaround.

### Phase 3: Discovery and subctl

**Files:** `submariner-operator/pkg/discovery/network/cilium.go`, `network.go`;
`subctl/pkg/diagnose/cilium.go`, `cni.go`; `cmd/subctl/join.go`.

Detection in the order given under Discovery and the cluster-pool fallback, so that the field, the
`--clustercidr` flag and the namespace key are no longer needed. Unit tests use fake DaemonSets and
ConfigMaps for cluster-pool, kubernetes and multi-pool IPAM, for chaining, and for the managed-distribution
markers. The join warning and the diagnose checks use the pod-exec helper in `internal/pods`; the OVN check
keeps its own exec code.

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

Phase 3:

- [ ] A Cilium cluster joins without the field, and `subctl diagnose cni` reports each prerequisite.

## Follow-ups

A second proposal, Globalnet service ingress without externalIPs, which I will submit after this one, moves
ClusterIP exports on Globalnet clusters into Submariner's own rules on the exporting gateway, the way
headless exports are handled today, and turns on by itself on Cilium clusters with kube-proxy replacement.
Clusters without Globalnet would need a small ingress range advertised to peers. In my kind runs a
release-0.25 peer did not pick up a subnet added to an existing Endpoint, and rejected an Endpoint that
arrived before its Cluster without re-checking it; I will file both as issues, and that part waits for those
fixes. A later step could give remote clusters a cluster-scoped Cilium identity, so that policy can select
them and Egress Gateway skips them; #4106 is the working reference for that.
