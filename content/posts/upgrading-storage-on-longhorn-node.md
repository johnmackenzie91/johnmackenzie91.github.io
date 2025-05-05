+++
title = "Upgrading Storage on Longhorn Node"
date = 2025-01-19T10:14:05Z
+++

One of my Longhorn nodes has ran out of storage.
I only have a 250GB SSD on it at the moment and need to upgrade it with a 1TB SSD.

I am going to remove the node from the cluster. Replace the SSD, readd the node to the cluster and begin replication.


Here are the steps that I took;

1. Drain the Node
Mark the node as unschedulable to prevent new workloads from being scheduled and drain existing workloads.
```bash
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

I did have to user the `--force` flag on the `drain` command as I was getting " Cannot evict pod as it would violate the pod's disruption budget.". This was because I only have a two node cluster.

Then verify that the node has drained;

```bash
kubectl get nodes
```

2. Remove the node from the cluster
Longhorn will detect the node’s removal and mark its replicas as Failed. Data will still be accessible as long as there are healthy replicas on other nodes.
```bash
kubectl delete node <node-name>
```

3. Remove current SSD and replace with new one
I first needed to ssh onto the node and remove the automatic mounting of the old SSD from the /etc/fstab file.
Once that was done I powered down the machine and switched out the HDD.
I then powered up. Formatted the  new HDD and ensure HDD is always mounted at startup with /etc/fstab.
4. Finally check that the machine is back on your network.
5. Re-Add the node to the cluster
This node is an agent. I am running the k3s install command as an agent.
```bash
curl -sfL https://get.k3s.io | \
  K3S_URL="<k3s_url>>" \
  K3S_TOKEN="<token>" \
  sh -s -   \
  --node-name agent-node-2   \
  --node-label workload=general   \
  --node-label storage=fast   \
  --data-dir /var/lib/k3s/agent
```
Wait and confirm that the node has returned to the cluster.
```bash
kubectl get nodes
```

6. In the Longhorn UI configure the new disk.
   * Open Longhorn UI.
   * Navigate to Node > Edit Node and Disks.
   * Add the new disk to the node by specifying the mount path (e.g., /mnt/longhorn-disk).
   * Mark the disk as Schedulable.
7. Verify that the data is getting replicated as expected.
8. Restore Scheduling to the node
```bash
kubectl uncordon <node-name>
```