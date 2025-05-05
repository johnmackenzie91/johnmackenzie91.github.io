+++
title = 'Migration of Longhorn Volume access modes'
date = 2024-01-24T10:14:05Z
+++

I am unable to change the access_mode of one of my Longhorn volumes from ReadWriteOnce to ReadWriteMany.
My plan is thus. To create another volume with the access_mode that I want.
Mount both volumes into a bash container and copy the data across.
Then mount this new volume in the place of the old volume. Wish me luck.

1. Create a New Volume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: some-data-2-persistent-volume-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: longhorn
```

2. Create a Temporary Pod
```bash
apiVersion: v1
kind: Pod
metadata:
  name: data-migration
spec:
  containers:
  - name: migration-container
    image: servercontainers/rsync:3.1.3
    command: ["rsync", "-ahv", "--progress", "--delete", "--ignore-existing", "/mnt/old/", "/mnt/new/"]
    volumeMounts:
    - mountPath: /mnt/old
      name: old-volume
    - mountPath: /mnt/new
      name: new-volume
  volumes:
  - name: old-volume
    persistentVolumeClaim:
      claimName: some-data-persistent-volume-claim
  - name: new-volume
    persistentVolumeClaim:
      claimName: some-data-2-persistent-volume-claim
```

This pod attach the two volumes, and uses rsync to move data from the old to the new.

4. Unmount and Update Deployment

Once the data copy is complete remove the pod. 
Then scale down  the deployment using the old volume to release the old PVC.
Update the deployment to use the new PVC with the ReadWriteMany volume.

5. Verify & Cleanup

### Important Considerations:

There will do downtime as we switch out the PVC. Not a problem for me and my homelab.
Also check that there is enough storage available. Start with having the new pvc with one replica and scale from there