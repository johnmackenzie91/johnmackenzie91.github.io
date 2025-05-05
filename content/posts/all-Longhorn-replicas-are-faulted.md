+++
title = 'All Longhorn replicas are faulted'
date = 2024-12-03T09:26:05Z
+++

Today my homelab has a total outage. All nodes went down together. Resilience is further down the line :grinning:.

In my longhorn UI it was saying that the Replicas were`Failed`.

I found the following solution brought back all of my longhorn replicas with data in tact;

1. Find the ID of the PVC that has failed
2. Stop all resources that are using that PVC.
3. Edit the `replicas.longhorn.io` resource for the relevant PVC ID

```
$ kubectl -n longhorn-system edit replicas.longhorn.io <PVC_ID>
```
4. Set the FailedAt field to be ""
5. Check the longhorn ui. You are expecting the PVC to have changed from `Failed` to `Detached`.
6. Bring up the previously stopped resources (deployments/pods/cronjobs ect)