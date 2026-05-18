+++
title = 'Fixing Private DNS Resolution in k3s: When Your Home Router Knows But CoreDNS Doesnt'
date = 2026-05-05T20:54:36+01:00
draft = false
+++

If you're running a homelab k3s cluster and have custom DNS entries on your home router — perhaps for a self-hosted service like Authentik — you may have run into this frustrating error:

```bash
dial tcp: lookup authentik.myexample.domain on 10.43.0.10:53: no such host
```

Your laptop resolves the domain just fine. Your phone resolves it. But pods inside your cluster can't. Here's exactly what's happening and how to fix it — including the k3s-specific gotcha that will undo a naive fix.
Understanding the DNS chain

When a pod needs to resolve a domain, it doesn't ask your router directly. It asks CoreDNS, the cluster-internal DNS server running at 10.43.0.10. CoreDNS then forwards the query upstream. The question is: upstream to where?

In a default k3s setup, CoreDNS is configured to forward using /etc/resolv.conf from whatever node it's running on. If that node's resolv.conf points to 1.1.1.1 or 8.8.8.8 instead of your router, your private DNS entries are simply invisible to the cluster.

The chain: Pod → CoreDNS (10.43.0.10) → node's /etc/resolv.conf → upstream DNS. If the upstream isn't your router, private entries are never found.
Step 1 — Confirm what CoreDNS is forwarding to

kubectl get configmap coredns -n kube-system -o yaml

Look for the forward line in the Corefile. You'll likely see:

```
forward . /etc/resolv.conf
```

This means CoreDNS is reading the upstream from the node's /etc/resolv.conf.
Step 2 — Check the node's resolv.conf

Find which node CoreDNS is running on:

```bash
kubectl get pods -n kube-system -o wide | grep coredns
```

SSH into that node and check:

```bash
cat /etc/resolv.conf
```

If you see nameserver 1.1.1.1 (Cloudflare) or 8.8.8.8 (Google) — there's your problem. CoreDNS is forwarding to public DNS, which has no knowledge of your private authentik.myexample.domain entry.

Ubuntu gotcha: On Ubuntu nodes, /etc/resolv.conf often points to 127.0.0.53 (systemd-resolved). CoreDNS can't reach that loopback address from inside the pod network, causing silent failures. Check with resolvectl status.
Step 3 — Verify your router can actually resolve it

Before changing anything, confirm the problem is CoreDNS config and not your router:

```bash
nslookup authentik.myexample.domain 192.168.1.1
```

Run this from the master node itself (not inside a pod). If this fails, your router's DNS config is the real issue.
The fix — and the k3s gotcha

The obvious approach is to edit the CoreDNS ConfigMap directly. Don't. Here's why.

k3s manages CoreDNS as a built-in Addon, annotated with:

objectset.rio.cattle.io/owner-gvk: k3s.cattle.io/v1, Kind=Addon

Any time k3s restarts or reconciles, it will silently overwrite your changes back to the default. You'll fix it, it'll work, then it'll mysteriously break again after a node reboot.

The right approach: k3s provides an intentional extension point via import /etc/coredns/custom/*.server in the Corefile. Use that instead.
The correct fix — ConfigMap approach (recommended)

The cleanest solution is a separate ConfigMap that k3s mounts into the CoreDNS pod's /etc/coredns/custom/ directory. It survives k3s upgrades, node replacements, and works correctly across multiple master nodes in future:

```bash
kubectl create configmap coredns-custom \
-n kube-system \
--from-literal=21fx.server='myexample.domain {
forward . 192.168.1.1
cache 30
}'
```

Then restart CoreDNS to pick it up:

```bash
kubectl rollout restart deployment/coredns -n kube-system
```

Why this works: This is a different ConfigMap from the one k3s manages. k3s won't reconcile it. The reload plugin in CoreDNS also picks up changes within ~30 seconds, so future edits don't always need a restart.
Alternative — drop a file on the node

If you prefer, you can place a file directly on the master node:

```bash
sudo mkdir -p /etc/coredns/custom
sudo nano /etc/coredns/custom/21fx.server
```

```bash
myexample.domain {
    forward . 192.168.1.1
    cache 30
}
```

This works, but you'd need to replicate it to every node if you ever add more masters — whereas the ConfigMap approach lives in etcd and is available cluster-wide automatically.
Test the fix

```bash
kubectl run dns-debug --image=busybox:1.28 --rm -it --restart=Never -- \
nslookup authentik.myexample.domain
```

You should now see a valid response from your router rather than no such host.