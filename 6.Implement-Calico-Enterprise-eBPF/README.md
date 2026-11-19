# Module 6: Implement Calico Enterprise eBPF

## In this lab

This lab provides the instructions to:

* [Overview](#overview)
* [Enable Calico Enterprise eBPF](#enable-calico-enterprise-ebpf)
* [Troubleshoot Calico Enterprise eBPF](#troubleshoot-calico-enterprise-ebpf)
* [Revert back to iptables dataplane](#revert-back-to-iptables-dataplane)

---

## Overview

Calico Enterprise has got a pluggable dataplane, which enables Calico Enterprise to easily implement and adopt various dataplanes. Calico Enterprise supports four dataplanes: the standard Linux iptables dataplane, the nftables dataplane, the eBPF dataplane, and the Windows dataplane. eBPF is an alternative dataplane for iptables in Linux. Instead of using Linux standard dataplane, which uses iptables and conntrack, the dataplane uses a set of bpf programs.
eBPF has lower overhead and better performance compared to iptables at large scales.

After finishing this lab, you learn how to enable Calico Enterprise eBPF along with some troubleshooting guidance.

---

## Enable Calico Enterprise eBPF

### 1. Verify a supported kernel and mounted BPF filesystem

Calico Enterprise eBPF requires Linux kernel v5.10 or later, or Red Hat kernel v4.18 with build number 305 or later (RHEL 8.4). SSH into all the cluster nodes and make sure they run a supported kernel. For more information, see the Calico Enterprise eBPF documentation:

https://docs.tigera.io/calico-enterprise/latest/operations/ebpf/enabling-ebpf

Run the following commands on each cluster node.

```bash
uname -rv
```

You should receive a similar output.

```text
6.8.0-1066-aws #69~22.04.1-Ubuntu SMP Mon Sep 21 16:20:15 UTC 2026
```

Verify that the BPF filesystem is mounted by running the following command.

```bash
mount | grep "/sys/fs/bpf"
```

You should receive a similar output.

```text
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
```

### 2. Find the API server address

In eBPF mode, Calico Enterprise implements Kubernetes service networking directly rather than relying on kube-proxy. This means that, like kube-proxy, Calico Enterprise needs to directly connect to the Kubernetes APIServer rather than via the APIServer’s ClusterIP. Refer to the following documentation for more information on this.

https://docs.tigera.io/calico-enterprise/latest/operations/ebpf/enabling-ebpf

First, make a note of the address of the APIServer. Run the following command to find the APIServer IP address.

```bash
kubectl get endpoints kubernetes -o wide
```

You should receive a similar output with a single IP address and port number.

```text
NAME         ENDPOINTS        AGE
kubernetes   10.0.1.20:6443   6d1h
```

### 3. Create the kubernetes-services-endpoint ConfigMap

Then, create the following config map in the tigera-operator namespace using the host and port number determined above.

```yaml
kubectl apply -f -<<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "10.0.1.20"
  KUBERNETES_SERVICE_PORT: "6443"
EOF
```

### 4. Verify the calico-system pods restart

It can take up to 60s for kubelet to pick up the ConfigMap. tigera operator will then do a rolling update of the relevant pods in the calico-system namespace to pass on the change. calico-node, calico-typha, and calico-kubecontroller pods should be restarted. Run the following command to validate that these pods are restarted. Make sure all the pods are back into fully Running state after the restart.

```bash
watch kubectl get pods -n calico-system
```

> [!NOTE]
> If you do not see the pods restart then it’s possible that the ConfigMap was not picked up (sometimes Kubernetes is slow to propagate ConfigMap ). Try restarting the tigera-operator.

### 5. Disable kube-proxy

In eBPF mode Calico Enterprise replaces kube-proxy and there is no need to run kube-proxy in the cluster. Run the following command to disable kube-proxy.

```bash
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
```

### 6. Enable eBPF mode in the Installation resource

To enable eBPF mode, change the `spec.calicoNetwork.linuxDataplane` parameter in the operator’s Installation resource to `BPF`.

```bash
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF"}}}'
```

> [!NOTE]
> This lab uses the default tunneled mode for traffic from outside the cluster to services. Calico Enterprise eBPF also supports Direct Server Return (DSR) mode, set with the `bpfExternalServiceMode` Felix configuration parameter. In DSR mode, the request from an external client is tunneled to the node running the backend pod, and the response goes directly from that node back to the client. This lowers latency, but the underlying network must allow it: on AWS, for example, the nodes must be in the same subnet. For more information, see https://docs.tigera.io/calico-enterprise/latest/operations/ebpf/enabling-ebpf.

### 7. Verify eBPF is enabled in the calico-node logs

Check calico-node pods logs to verify eBPF is enabled.

```bash
for i in $(kubectl get pods -n calico-system -o name | grep calico-node) ; do kubectl logs $i -n calico-system | grep -w "BPF enabled" ; done
```

You should see an output similar to the following.

```text
2026-10-10 05:28:36.405 [INFO][56302] felix/int_dataplane.go 883: BPF enabled, configuring iptables/nftables layer to clean up kube-proxy's rules.
2026-10-10 05:28:36.696 [INFO][56302] felix/int_dataplane.go 1522: BPF enabled, starting BPF endpoint manager and map manager.
2026-10-10 05:28:36.814 [INFO][56302] felix/int_dataplane.go 3613: BPF enabled, disabling unprivileged BPF usage.
2026-10-10 05:28:52.520 [INFO][57188] felix/int_dataplane.go 883: BPF enabled, configuring iptables/nftables layer to clean up kube-proxy's rules.
2026-10-10 05:28:52.648 [INFO][57188] felix/int_dataplane.go 1522: BPF enabled, starting BPF endpoint manager and map manager.
2026-10-10 05:28:52.707 [INFO][57188] felix/int_dataplane.go 3613: BPF enabled, disabling unprivileged BPF usage.
2026-10-10 05:29:44.193 [INFO][54837] felix/int_dataplane.go 883: BPF enabled, configuring iptables/nftables layer to clean up kube-proxy's rules.
2026-10-10 05:29:44.747 [INFO][54837] felix/int_dataplane.go 1522: BPF enabled, starting BPF endpoint manager and map manager.
2026-10-10 05:29:44.966 [INFO][54837] felix/int_dataplane.go 3613: BPF enabled, disabling unprivileged BPF usage.
```

-----

### 8. Validate eBPF is enforced on the interfaces

Let's validate if eBPF is enforced. Calico Enterprise attaches its eBPF programs to each network interface through the kernel's traffic control (`tc`) hook, using a `clsact` qdisc that gives it an ingress and an egress attachment point on every interface. You list them with `bpftool`. The validation can be done directly from the cluster node by SSHing into the node or via the calico-node pod. Let's do the validation via `control1` node and the `calico-node` pod running on `control1`.

**Validation from cluster node**

ssh into `control1` node.

```bash
ssh control1
```

`bpftool` is not installed on the lab nodes by default. On Ubuntu it ships in the `linux-tools` package that matches the running kernel, so install it with the following commands. `$(uname -r)` picks the package for the node's kernel, for example `linux-tools-5.15.0-1084-aws`.

```bash
sudo apt-get install -y linux-tools-common linux-tools-$(uname -r)
```

Run the following command.

```bash
sudo bpftool net show
```

You should see an output similar to the following. Each line under `tc:` shows one interface and one Calico eBPF program attached to it:

* `ens5` is the node's network card, and each `cali*` interface connects one pod to the node.
* `cali_tc_preambl` is Calico's entry program (`bpftool` truncates the name `cali_tc_preamble`). Every packet goes through it first, and it passes the packet on to the rest of Calico's eBPF code.
* `clsact/ingress` catches packets coming into the interface, and `clsact/egress` catches packets going out. Each interface has both.
* The number in brackets and after `id` is the kernel's ID for the loaded eBPF program.

> [!NOTE]
> On kernels 6.6 and newer, Calico uses the newer TCX attachment point instead, and these lines read `tcx/ingress` and `tcx/egress`.

This is why eBPF mode works: Calico sees every packet entering or leaving the node and its pods, so it can apply network policy and load-balance services itself, without iptables or kube-proxy.

```text
xdp:

tc:
lo(1) clsact/ingress cali_tc_preambl:[956] id 956
lo(1) clsact/egress cali_tc_preambl:[958] id 958
ens5(2) clsact/ingress cali_tc_preambl:[1004] id 1004
ens5(2) clsact/egress cali_tc_preambl:[998] id 998
cali349113e2681(3) clsact/ingress cali_tc_preambl:[1058] id 1058
cali349113e2681(3) clsact/egress cali_tc_preambl:[1064] id 1064
calibd567c90380(6) clsact/ingress cali_tc_preambl:[1040] id 1040
calibd567c90380(6) clsact/egress cali_tc_preambl:[1041] id 1041
cali8b5d1a98c5f(11) clsact/ingress cali_tc_preambl:[1051] id 1051
cali8b5d1a98c5f(11) clsact/egress cali_tc_preambl:[1053] id 1053
calia28da0d09ff(18) clsact/ingress cali_tc_preambl:[1065] id 1065
calia28da0d09ff(18) clsact/egress cali_tc_preambl:[1063] id 1063
calie702a04113a(19) clsact/ingress cali_tc_preambl:[1031] id 1031
calie702a04113a(19) clsact/egress cali_tc_preambl:[1034] id 1034
calib68cf938cfb(22) clsact/ingress cali_tc_preambl:[1035] id 1035
calib68cf938cfb(22) clsact/egress cali_tc_preambl:[1029] id 1029
cali7fd6f2bf18c(23) clsact/ingress cali_tc_preambl:[1047] id 1047
cali7fd6f2bf18c(23) clsact/egress cali_tc_preambl:[1049] id 1049
egress.calico(41) clsact/ingress cali_tc_preambl:[1002] id 1002
egress.calico(41) clsact/egress cali_tc_preambl:[1000] id 1000
bpfout.cali(56) clsact/ingress cali_tc_preambl:[983] id 983
bpfout.cali(56) clsact/egress cali_tc_preambl:[981] id 981

flow_dissector:
```

Exit the ssh session.

```bash
exit
```

**Validation from calico-node**

Run the following command to do the validation from calico-node running on `control1`.

```bash
CALICO_NODE=$(kubectl get pods -n calico-system -l k8s-app=calico-node -o wide | grep 10.0.1.20 | awk {'print $1'}) && echo $CALICO_NODE
kubectl exec -ti $CALICO_NODE -n calico-system -c calico-node -- bpftool net show
```

You should see the same output as on the node.

### 9. Inspect the eBPF configuration with the Calico BPF tool

Calico Enterprise includes a BPF tool to interact with Calico Enterprise eBPF and collect relevant information. It is built into the `calico-node` binary in the calico-node pods, and you run it as `calico-node -bpf <command>`. Let's use it to view the configurations implemented by eBPF. We will run these commands from the calico-node pod running on `control1`.

```bash
CALICO_NODE=$(kubectl get pods -n calico-system -l k8s-app=calico-node -o wide | grep 10.0.1.20 | awk {'print $1'}) && echo $CALICO_NODE
```

```bash
kubectl exec -ti $CALICO_NODE -n calico-system -c calico-node -- calico-node -bpf help
```

> [!NOTE]
> In the latest Calico Enterprise releases, the BPF tool is run as `calico component node bpf <command>` instead of `calico-node -bpf <command>`.

You should see an output similar to the following.

```text
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), cni-plugins (init), install-cni (init)
tool for interrogating Calico BPF state

Usage:
  calico-bpf [command]

Available Commands:
  arp          Manipulates arp
  cleanup      Removes all calico-bpf programs and maps
  completion   Generate the autocompletion script for the specified shell
  connect-time Manipulates connect-time load balancing programs
  conntrack    Manipulates connection tracking
  counters     Show and reset counters
  help         Help about any command
  ifstate      Manipulates ifstate
  ipsets       Manipulates ipsets
  nat          Manipulates network address translation (nat)
  policy       Dump policy attached to interface
  profiling    Show and reset profiling data
  routes       Manipulates routes
  version      Prints the version and exits

Flags:
      --config string      config file (default is $HOME/.calico-bpf.yaml)
  -h, --help               help for calico-bpf
  -6, --ipv6               Use IPv6 instead of IPv4
  -j, --json               Output in JSON format
      --log-level string   Set log level (default "warn")

Use "calico-bpf [command] --help" for more information about a command.
```

### 10. Show the Linux routing table on control1

Calico Enterprise eBPF routes packets before reaching Linux routing tables. However, Calico Enterprise eBPF always uses Linux routing table to route the first packet and take advantage of Linux kernel security features, RPF, etc. Calico Enterprise eBPF records this information as a conntrack entry in the conntrack map and skip all the processing for the subsequent flows.

SSH into `control1` and show the routing table.

```bash
ssh control1
```

```bash
ip route show
```

You should see an output similar to the following.

```text
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20 metric 100
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100
10.48.0.0 dev cali349113e2681 scope link metric 1024
blackhole 10.48.0.0/26 proto bird
10.48.0.3 dev calibd567c90380 scope link metric 1024
10.48.0.4 dev cali8b5d1a98c5f scope link metric 1024
10.48.0.5 dev calia28da0d09ff scope link metric 1024
10.48.0.6 dev calie702a04113a scope link metric 1024
10.48.0.7 dev calib68cf938cfb scope link metric 1024
10.48.0.8 dev cali7fd6f2bf18c scope link metric 1024
10.48.0.64/26 via 10.0.1.30 dev ens5 proto bird metric 1024
10.48.0.192/26 via 10.0.1.31 dev ens5 proto bird metric 1024
10.48.2.224/29 via 10.0.1.30 dev ens5 proto bird metric 1024
10.48.128.64/26 via 10.0.1.30 dev ens5 proto bird metric 1024
10.48.128.192/26 via 10.0.1.31 dev ens5 proto bird metric 1024
10.49.0.10 via 169.254.1.1 dev bpfin.cali src 10.0.1.20
169.254.1.1 dev bpfin.cali scope link
```

A few things worth noting:

* **The `bpfin.cali` routes only exist in eBPF mode.** `bpfin.cali` and `bpfout.cali` are a veth pair Calico creates in the host namespace. Traffic that originates on the node itself, such as DNS lookups to the kube-dns ClusterIP `10.49.0.10`, is steered through this pair so Calico's eBPF programs can do the service NAT, since kube-proxy and its iptables rules are gone.
* **The `blackhole 10.48.0.0/26` route is this node's own IPAM block.** Packets to an address in the block that has no pod are dropped here rather than forwarded, which prevents routing loops.
* **The rest of the table is unchanged by eBPF.** Everything you learned in the earlier modules still applies. Each local pod has a `/32` route pointing at its `cali*` interface. The `/26` blocks learned over BGP (`proto bird`) show the IPAM blocks that worker1 (`10.0.1.30`) and worker2 (`10.0.1.31`) own.

Exit the ssh session.

```bash
exit
```

### 11. Dump the eBPF routing table

Now, let's check the eBPF routing table on calico-node running on `control1`.

```bash
kubectl exec -ti $CALICO_NODE -n calico-system -c calico-node -- calico-node -bpf routes dump
```

You should see an output similar to the following.

```text
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), cni-plugins (init), install-cni (init)
   10.0.1.20/32: local host
   10.0.1.30/32: remote host
   10.0.1.31/32: remote host
   10.48.0.0/24: remote in-pool nat-out
   10.48.0.0/26: remote workload in-pool nat-out nh 10.0.1.31
 10.48.0.128/32: local workload in-pool nat-out idx 4
 10.48.0.132/32: local workload in-pool nat-out idx 12
 10.48.0.133/32: local workload in-pool nat-out idx 13
 10.48.0.134/32: local workload in-pool nat-out idx 14
 10.48.0.135/32: local workload in-pool nat-out idx 15
 10.48.0.136/32: local workload in-pool nat-out idx 18
 10.48.0.137/32: local workload in-pool nat-out idx 19
 10.48.0.138/32: local workload in-pool nat-out idx 20
 10.48.0.192/26: remote workload in-pool nat-out nh 10.0.1.30
  172.17.0.1/32: local host
```

Note the following about the above output.

* `10.0.1.20/32: local host` = ip is the local node ip
* `10.0.1.30/32: remote host` = ip is a remote node ip
* `10.48.162.128/26: remote workload in-pool nat-out nh 10.0.1.31` = remote pod network on node -> 10.0.1.31
* `10.255.241.69/32: local workload in-pool nat-out idx 77` = local pod

### 12. List the summary service and its endpoints

As discussed before, in eBPF mode, Calico Enterprise implements Kubernetes service networking directly rather than relying on kube-proxy. Let's check the service implementation details through eBPF.

Run the following commands to list yaobank `summary` service and endpoints. This service has two endpoints. We will see how Calico Enterprise eBPF implements this service next.

```bash
kubectl get svc -n yaobank summary
```

**Example output:**

```text
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
summary   ClusterIP   10.49.18.111   <none>        80/TCP    6m4s
```

Save the service's cluster IP in a variable. You will use it in the next step.

```bash
SUMMARY_IP=$(kubectl get svc -n yaobank summary -o jsonpath='{.spec.clusterIP}') && echo $SUMMARY_IP
```

**Example output:**

```text
10.49.18.111
```

```bash
kubectl get endpoints -n yaobank summary
```

**Example output:**

```text
NAME      ENDPOINTS                       AGE
summary   10.48.0.210:80,10.48.0.212:80   9m58s
```

### 13. View the service NAT implementation in eBPF

Run the following command to list the `summary` service implemented through eBPF. It uses the cluster IP you saved in `SUMMARY_IP` in the previous step.

```bash
kubectl exec -ti $CALICO_NODE -n calico-system -c calico-node -- calico-node -bpf nat dump | grep -A2 "$SUMMARY_IP port"
```

You should see an output similar to the following.

```text
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), cni-plugins (init), install-cni (init)
10.49.18.111 port 80 proto 6 id 23 count 2 local 0
        23:0     10.48.0.210:80
        23:1     10.48.0.212:80
```

Find below the explanation of each field in the previous output.

    [service ip] [service port] [protocol number] [id] [count] [local]
    [id:countid] [pod ip:port]

> [service ip] - kubernetes clusterIP/externalIp/nodeIP
>
> [service port] -  kubernetes - Service port
>
> [protocol number] - Protocol number used
>
> [id] - identification number (unique)
>
> [count] - number of pods on this service
>
> [local] - how many pods are local on current node
>
> [id:countid] - “identification number”:”each pod ip have assigned id”
>
> [pod ip:port]

### 14. Create a NetworkPolicy for the customer pod

Before using the Calico BPF tool to find policies implemented for an interface, let's deploy a policy for a pod. Three rules implemented in this policy for the sake of demonstration.

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.customer
  namespace: yaobank
  labels:
    projectcalico.org/tier: default
spec:
  tier: default
  order: 100
  selector: app == "customer"
  ingress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        ports:
          - '80'
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        ports:
          - '443'
    - action: Allow
      protocol: ICMP
      source: {}
      destination: {}
  types:
    - Ingress
EOF
```

### 15. Find the customer pod, its node, and its calico-node pod

Find the customer pod's IP address and the node it is running on, and save them in variables. Policy for a pod is enforced on the node where the pod runs, so you also save the name of the calico-node pod on that node. You will use these variables in the next steps.

```bash
kubectl get pods -n yaobank -l app=customer -o wide
```

**Example output:**

```text
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE                                         NOMINATED NODE   READINESS GATES
customer-77b9fd549b-kxgbh   1/1     Running   0          67m   10.48.0.218   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
```

```bash
CUSTOMER_IP=$(kubectl get pods -n yaobank -l app=customer -o jsonpath='{.items[0].status.podIP}') && echo $CUSTOMER_IP
CUSTOMER_HOST=$(kubectl get pods -n yaobank -l app=customer -o jsonpath='{.items[0].spec.nodeName}') && echo $CUSTOMER_HOST
CUSTOMER_CALICO_NODE=$(kubectl get pods -n calico-system -l k8s-app=calico-node --field-selector spec.nodeName=$CUSTOMER_HOST -o jsonpath='{.items[0].metadata.name}') && echo $CUSTOMER_CALICO_NODE
```

**Example output:**

```text
10.48.0.218
ip-10-0-1-31.ca-central-1.compute.internal
calico-node-9vp7n
```

### 16. Find the pod interface name

Each pod is connected to its node by a `cali*` interface, and the pod's policy is attached to that interface. Ask the node which interface leads to the pod's IP. The calico-node pod uses the node's network, so you can run the command from there without SSHing into the node.

```bash
CUSTOMER_IFACE=$(kubectl exec -n calico-system $CUSTOMER_CALICO_NODE -c calico-node -- ip route get $CUSTOMER_IP | awk '/dev/ {print $3}') && echo $CUSTOMER_IFACE
```

**Example output:**

```text
cali477407eea3b
```

### 17. Dump the policy attached to the pod interface

Now run the following command. It uses the calico-node pod and the interface name you saved in the previous two steps.

> [!NOTE]
> The command is `kubectl exec -ti <calico-node pod> -n calico-system -- calico-node -bpf policy dump <interface> <hook>`. The interface is the host endpoint (HEP) or workload endpoint (WEP) interface name. `<hook>` can be `ingress`, `egress`, `xdp` or `all`. Add `--asm` to also print the eBPF assembler code of the policy program.

```bash
kubectl exec -ti $CUSTOMER_CALICO_NODE -n calico-system -c calico-node -- calico-node -bpf policy dump $CUSTOMER_IFACE ingress
```

You should see an output similar to the following. Get yourself familiar with the following output. Note how calico bpf till list the policy rules according to the order they were implemented in the policy above.

```text
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), cni-plugins (init), install-cni (init)
IfaceName: cali477407eea3b
Hook: tc egress
  Tier: default
    Policy: NetworkPolicy yaobank/default.customer 0
      Rule: yaobank/default.customer  Action: allow
      Hit count: 0
      Rule: yaobank/default.customer  Action: allow
      Hit count: 0
      Rule: yaobank/default.customer  Action: allow
      Hit count: 0
    End Policy: NetworkPolicy yaobank/default.customer 0
  End Tier: default  (action: deny)
      Rule: kns.yaobank  Action: allow
      Hit count: 0
```

### 18. Watch the policy rule counts change under load

Record the `Hit count` for each rule above, then send some curl requests to yaobank and see how the counts change. The `customer` service is exposed as a NodePort, so send the requests from the bastion to port `30180` on `control1`, as you did in earlier modules.

```bash
for i in 1 2 3 4; do curl -s -o /dev/null -w '%{http_code}\n' http://10.0.1.20:30180; done
```

The hit count goes up once per new connection, not once per packet. Calico Enterprise checks policy only on the first packet of a connection; after that, the connection is tracked and its packets are allowed without checking the policy again. Each `curl` command above opens a new TCP connection to the customer pod, so the 4 requests add 4 hits to the port 80 rule.

Run the policy dump again.

```bash
kubectl exec -ti $CUSTOMER_CALICO_NODE -n calico-system -c calico-node -- calico-node -bpf policy dump $CUSTOMER_IFACE ingress
```

**Example output:**

```text
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), cni-plugins (init), install-cni (init)
IfaceName: cali477407eea3b
Hook: tc egress
  Tier: default
    Policy: NetworkPolicy yaobank/default.customer 0
      Rule: yaobank/default.customer  Action: allow
      Hit count: 4
      Rule: yaobank/default.customer  Action: allow
      Hit count: 0
      Rule: yaobank/default.customer  Action: allow
      Hit count: 0
    End Policy: NetworkPolicy yaobank/default.customer 0
  End Tier: default  (action: deny)
      Rule: kns.yaobank  Action: allow
      Hit count: 5
```

Alternatively, if you want to see more information about your policy, run the following command to see the port and protocol information.

```bash
kubectl exec -n calico-system $CUSTOMER_CALICO_NODE -c calico-node -- \
  calico-node -bpf policy dump $CUSTOMER_IFACE ingress --asm \
  | grep -E 'Hook:|Policy:|Rule:|Hit count|// If ' | grep -v 'named ports {}'
```

```text
Hook: tc egress
    Policy: NetworkPolicy yaobank/default.customer 0
      Rule: yaobank/default.customer  Action: allow
      Hit count: 4
      // If protocol != tcp, skip to next rule
      // If dest port is not within any of {80}, skip to next rule
      Rule: yaobank/default.customer  Action: allow
      Hit count: 0
      // If protocol != tcp, skip to next rule
      // If dest port is not within any of {443}, skip to next rule
      Rule: yaobank/default.customer  Action: allow
      Hit count: 0
      // If protocol != icmp, skip to next rule
    End Policy: NetworkPolicy yaobank/default.customer 0
      Rule: kns.yaobank  Action: allow
      Hit count: 4
```

### 19. List the ipsets in use on the node

Network policies select pods by label, for example `app == "customer"`. Packets don't carry labels, though: a packet only has a source and destination IP address and port. So when a packet arrives, Calico's eBPF program needs a way to know whether the packet's IP belongs to a pod with those labels. Calico Enterprise solves this with IP sets: for each selector used in a policy, Felix keeps a list of the IPs of the pods that currently match it. A policy rule then checks "is the source (or destination) in IP set X?" instead of listing pod IPs.

This makes policy fast and easy to keep up to date:

* When a pod starts or stops, Felix only adds or removes its IP in the set. The policy program itself doesn't change.
* Checking a set is a single lookup, whether the selector matches one pod or thousands.

In eBPF mode, all IP sets on a node are stored in one BPF map. To list them, run the following command.

```bash
kubectl exec -ti $CUSTOMER_CALICO_NODE -n calico-system -c calico-node -- calico-node -bpf ipsets dump
```

You should see an output similar to the following. However, the list can be different depending on the available ipsets on the node.

```text
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), cni-plugins (init), install-cni (init)
IP set 0x100000001
   10.49.0.10:53 (proto 17)

IP set 0xd26fa31f8d497f4
   10.48.0.135/32
   10.48.0.208/32

IP set 0x3daf74135d5cb3e9
   10.0.1.20/32

IP set 0x6b040aa4ed99b698
   10.48.0.194/32

IP set 0x6e77fb5b61e41fed
   10.48.0.2/32
   10.48.0.5/32

IP set 0x7edc0e52b1c7a2ac
   10.48.0.11/32
   10.48.0.199/32

IP set 0x9a85ddf3b2bb9365
   10.0.1.20:6443 (proto 6)

IP set 0xa94ca2b47cd0c227
   10.48.0.197/32

IP set 0xae04d0ae61bb92ee
   10.48.0.137:5443 (proto 6)
   10.48.0.137:8080 (proto 6)
   10.48.0.14:5443 (proto 6)
   10.48.0.14:8080 (proto 6)

IP set 0xca2fa4f94dc261c4
   10.48.0.201/32

IP set 0xdfdd106f994fdb54
   10.48.0.197/32
```

How to read the output:

* `IP set 0x...` is the ID of the set. Felix derives it from the selector the set represents.
* Each line under it is a member: one pod IP (`/32`), or a CIDR from a network set or a rule that matches on `nets`.
* Some members also have a port and protocol, such as `10.49.0.10:53 (proto 17)`, where 17 is UDP and 6 is TCP. These come from rules that match named ports or Kubernetes services.
* `IP set 0x100000001` is a reserved set: the trusted DNS servers, here the `kube-dns` service. Calico Enterprise reads DNS replies only from these servers, which is how policies that allow traffic by domain name learn the IPs behind those names.

### 20. Dump the conntrack map

To list the conntrack map, run the following command.

```bash
kubectl exec -ti $CALICO_NODE -n calico-system -c calico-node -- calico-node -bpf conntrack dump
```

You should see an output similar to the following. The following output is truncated.

```text
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), ebpf-bootstrap (init), cni-plugins (init), install-cni (init)
ConntrackKey{proto=17 10.48.0.2:53 <-> 10.48.0.135:36450} -> Entry{Type:0, Created:13280597154143, LastSeen:13280597444227, Flags: 0x110 B-A Data: {A2B:{Bytes:225 Packets:1 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:false Ifindex:2} B2A:{Bytes:132 Packets:1 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:true Ifindex:15} OrigDst:0.0.0.0 OrigSrc:0.0.0.0 OrigPort:0 OrigSPort:0 TunIP:0.0.0.0}} Age: 24.318178817s Active ago 24.317888733s
ConntrackKey{proto=6 10.0.1.20:56738 <-> 169.254.169.254:80} -> Entry{Type:0, Created:13297805187841, LastSeen:13297805741251, Flags: <none> Data: {A2B:{Bytes:616 Packets:5 Seqno:1044308989 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:false Opener:true Ifindex:0} B2A:{Bytes:585 Packets:5 Seqno:2967668612 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:true Opener:false Ifindex:2} OrigDst:0.0.0.0 OrigSrc:0.0.0.0 OrigPort:0 OrigSPort:0 TunIP:0.0.0.0}} Age: 7.110195217s Active ago 7.109641807s CLOSED
ConntrackKey{proto=17 10.48.0.5:53 <-> 10.48.0.132:13963} -> Entry{Type:0, Created:13256085471314, LastSeen:13256086882425, Flags: 0x110 B-A Data: {A2B:{Bytes:677 Packets:3 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:false Ifindex:2} B2A:{Bytes:407 Packets:3 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:true Ifindex:12} OrigDst:0.0.0.0 OrigSrc:0.0.0.0 OrigPort:0 OrigSPort:0 TunIP:0.0.0.0}} Age: 48.829949311s Active ago 48.8285382s
```

---

## Troubleshoot Calico Enterprise eBPF

Troubleshooting eBPF at time could require enabling `debug` logging. In this section, we will learn how to enable `debug` logging for eBPF.

### 1. Enable Felix debug logging for eBPF

Set Felix `bpfLogLevel` to `Debug`. Note that enabling debug logging causes a firehose of logs to the BPF trace pipe and should only be activated once requested by Tigera support team. Run the following command to enable debug logging.

```bash
kubectl patch felixconfiguration default --patch='{"spec": {"bpfLogLevel": "Debug"}}'
```

### 2. Find the calico-node pod on worker1

The logs are written to the kernel trace buffer of each node. You read them with `bpftool`, which is included in the calico-node pods, so you don't need to SSH into the node. Find the calico-node pod running on `worker1`.

```bash
WORKER1_NODE=$(kubectl get pods -n calico-system -l k8s-app=calico-node -o wide | grep 10.0.1.30 | awk {'print $1'}) && echo $WORKER1_NODE
```

### 3. View the BPF trace logs

Run the following command to view the logs. Press `ctrl+c` once done. Note that every decision point for every packet is logged!

```bash
kubectl exec -ti $WORKER1_NODE -n calico-system -c calico-node -- bpftool prog tracelog
```

The first time you run it, `bpftool` may print `could not find tracefs, attempting to mount it now` before the logs start.

You should see firehose of output similar to the following.

`<...>-84582 [000] .Ns1  6851.690474: 0: ens192---E: Final result=ALLOW (-1). Program execution time: 7366ns`

Following is a short description of the output above.

> [84582] - PID that triggered NAPI poll (often useful)
>
> [000] - CPU number
>
>[6851.690474] - Timestamp
>
>[0:] - Calico program ID then
>[ens192] - Interface caliXXX, ethXX
>[---E:] - E=egress, I=ingress, C=connect-time
>
>[Final result=ALLOW (-1). Program execution time: 7366ns] - Message
>

### 4. Turn off debug logging

When you're done, set `bpfLogLevel` back to its default, `Off`.

```bash
kubectl patch felixconfiguration default --patch='{"spec": {"bpfLogLevel": "Off"}}'
```

---

## Revert back to iptables dataplane

To revert back to standard Linux dataplane, follow the below procedure.

### 1. Revert the Installation resource to the iptables dataplane

Revert the changes to the operator’s Installation resource.

```bash
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"Iptables"}}}'
```

### 2. Verify the calico-node pods restart

calico-node pods should get restarted. Run the following command to validate the calico-node restart.

```bash
watch kubectl get pods -n calico-system
```

### 3. Re-enable kube-proxy

If you disabled kube-proxy, re-enable it.

```bash
kubectl patch ds -n kube-system kube-proxy --type merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": null}}}}}'
```

### 4. Verify kube-proxy pods are Running

Make sure kube-proxy pods are in Running state.

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

**Example output:**

```text
NAME               READY   STATUS    RESTARTS   AGE
kube-proxy-87lzp   1/1     Running   0          41s
kube-proxy-bgb8t   1/1     Running   0          41s
kube-proxy-gzvsb   1/1     Running   0          41s
```

Check that the Calico eBPF programs are removed from the interfaces. ssh into `control1` and run `bpftool` again.

```bash
ssh control1
sudo bpftool net show
```

The `tc:` section should no longer list any `cali_tc_preambl` programs. Exit the ssh session.

```bash
exit
```

### 5. Restart pods to recover long-lived connections

Switching from `eBPF` to `iptables` dataplane is disruptive. Long-lived connections can't be recovered unless the application pods are reconnected on their own, preventing a restart. Otherwise, these pods need to be restart to recover long living connections. Use the following command to restart all the pods.

```bash
 kubectl delete pods -A --all --force --grace-period=0
```

> **Congratulations! You have completed `6. Implement Calico Enterprise eBPF` lab.**

---

[← Previous: 5. Calico Enterprise Egress Gateway](../5.Calico-Enterprise-Egress-Gateway/README.md) | [Training overview](../README.md)
