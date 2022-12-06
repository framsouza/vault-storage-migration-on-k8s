# Migrating vault storage backend on Kubernetes with Helm

Are you struggling to migrate your vault storage backend or even performing a seal migration when running vault on top of Kubernetes? If so, this page is for you.

Recently I saw myself in this situation and while googling around I realized there's no guide on how to do that when you are running vault workloads on top of Kubernetes, which makes things a little tricky to deal with.

There are a set of manual steps that need to be performed, so bear with me.

## Context
Just for a matter of context, I ended up in a situation where I had to perform:

- vault storage migration from an external source (AWS) to Vault Integration Storage (AKA raft)
- Seal migration from Sharmir to Cloud auto unseal (GCP)

The vault version that had been used was quite old, as well as the Kubernetes version.

With that in mind, the migration plan was created.

## Migration plan

As I mentioned before, you need to perform some manual work (I would dedicate more time to automate that but spare time is hard to find).

Let's assume you are using helm to deploy vault on Kubernetes and you are using the [values.yml](values.yaml). As you can see in the file, I'm using GCPKMS to auto unseal vault and the vault path where raft is going to store data is **/vault/data**. This is the configuration you need for the new vault instance.

Once the pods are up and running, make sure you scale down the pods to only 1 (it will avoid leader election confusion once the migration is finished), `kubectl scale -n vault sts/vault --replicas=1`.

If you look closer at the manifest, there's no liveness and readiness probe, but in case your statefulset has any of these settings, make sure to remove it, otherwise, the pods will restart all the time and you won't be able to finish the migration.

At this point, you should have one vault instance open and running. 

As part of the storage migration, [hashiCorp guides](https://developer.hashicorp.com/vault/tutorials/raft/raft-migration) us to create a _migrate.hcl_ file pointing to source and destination storage.

Let's add this migrate.hcl as part of our vault statefulset:

1. Create a _migrate.hcl_ with the following content:
```
storage_source "gcs" {
  bucket = "my-vault-bucket"
}

storage_destination "raft" {
  path = "/vault/data"
}

cluster_addr = "http://$(HOSTNAME).vault-internal:8201"
```
Make sure to adjust the source bucket to the correct name, also, make sure on the destination you are using the same path as you have configured in your raft settings (**/vault/data**).

2. Create a configmap using the migrate file as a source, `kubectl create configmap -n vault migrate --from-file=migrate.hcl`
3. Edit your statefulset and mount the configmap as a volume:
```
      volumes:
      - configMap:
          defaultMode: 420
          name: migrate
        name: migrate
...
        volumeMounts:
        - mountPath: /tmp/migrate/
          name: migrate
```
Pay attention to the volume and configMap name, make sure they are correct. This is going to mount the _migrate.hcl_ on _/tmp/migrate/migrate.hcl_ inside the pod.

Now is here the fun begins:

4. Edit your statefulset and replace the line below:
```
      containers:
      - args:
        - "cp /vault/config/extraconfig-from-values.hcl /tmp/storageconfig.hcl;\n[
          -n \"${HOST_IP}\" ] && sed -Ei \"s|HOST_IP|${HOST_IP?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${POD_IP}\" ] && sed -Ei \"s|POD_IP|${POD_IP?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${HOSTNAME}\" ] && sed -Ei \"s|HOSTNAME|${HOSTNAME?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${API_ADDR}\" ] && sed -Ei \"s|API_ADDR|${API_ADDR?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${TRANSIT_ADDR}\" ] && sed -Ei \"s|TRANSIT_ADDR|${TRANSIT_ADDR?}|g\"
          /tmp/storageconfig.hcl;\n[ -n \"${RAFT_ADDR}\" ] && sed -Ei \"s|RAFT_ADDR|${RAFT_ADDR?}|g\"
          /tmp/storageconfig.hcl;\n/usr/local/bin/docker-entrypoint.sh vault server
          -config=/tmp/storageconfig.hcl \n"
        command:
        - /bin/sh
        - -ec
```

By:

```
      containers:
      - command:
        - /bin/sh
        - /usr/local/bin/docker-entrypoint.sh
        - sleep
        - 10d
```

You might be asking why you need to do that.
Well, vault already has a state, if you try to run a migration without deleting the current vault data, you will get the following error:
```
kubectl logs vault-0 -n vault -f
Error migrating: error mounting 'storage_destination': could not bootstrap clustered storage: error bootstrapping cluster: cluster already has state
```

Once you edited the statefulset, make sure the pod was recreated to place the new configuration.

5. Access the vault pod and delete the current state, 
- `kubectl exec -ti vault-acc-0 -n vault-acc -- sh`
- `$ cd /vault/data && rm -rf *`

**PRO TIP**: If the migration already locked the bucket, you will probably get an error when you try to perform the migration again, if that happens (I would recommend running that anyway) run from inside the pod: `$ vault operator migrate -config /tmp/migrate/migrate.hcl -reset`

6. Once you deleted the current state, let's edit the statefulset again and ask that to perform the real migration. Replace the statefulset commands following the instructions below:

```
      containers:
      - command:
        - /bin/sh
        - /usr/local/bin/docker-entrypoint.sh
        - sleep
        - 10d
```

BY

```
      containers:
      - command:
        - /bin/sh
        - /usr/local/bin/docker-entrypoint.sh
        - vault
        - operator
        - migrate
        - -config
        - /tmp/migrate/migrate.hcl
```

This is going to instruct vault to read the migration file and start the migration. Recreate the pod, and check the logs, you will probably see something similar to that:
```
kubectl logs vault-0 -n vault-acc -f
2022-12-06T09:22:44.439Z [INFO]  creating Raft: config="&raft.Config{ProtocolVersion:3, HeartbeatTimeout:5000000000, ElectionTimeout:5000000000, CommitTimeout:50000000, MaxAppendEntries:64, BatchApplyCh:true, ShutdownOnRemove:true, TrailingLogs:0x2800, SnapshotInterval:120000000000, SnapshotThreshold:0x2000, LeaderLeaseTimeout:2500000000, LocalID:\"fd1891e7-f066-1c73-3d80-544b9468cb45\", NotifyCh:(chan<- bool)(0xc000115c70), LogOutput:io.Writer(nil), LogLevel:\"DEBUG\", Logger:(*hclog.intLogger)(0xc0001e8120), NoSnapshotRestoreOnStart:true, skipStartup:false}"
2022-12-06T09:22:44.440Z [INFO]  initial configuration: index=1 servers="[{Suffrage:Voter ID:fd1891e7-f066-1c73-3d80-544b9468cb45 Address:$(HOSTNAME).vault-acc-internal:8201}]"
2022-12-06T09:22:44.440Z [INFO]  entering follower state: follower="Node at fd1891e7-f066-1c73-3d80-544b9468cb45 [Follower]" leader=
2022-12-06T09:22:52.091Z [WARN]  heartbeat timeout reached, starting election: last-leader=
2022-12-06T09:22:52.091Z [INFO]  entering candidate state: node="Node at fd1891e7-f066-1c73-3d80-544b9468cb45 [Candidate]" term=2
2022-12-06T09:22:52.096Z [INFO]  election won: tally=1
2022-12-06T09:22:52.529Z [INFO]  copied key: path=audit/3e9c4600-6b31-1b44-as21-289a835c3d8b/test
2022-12-06T09:22:52.669Z [INFO]  copied key: path=auth/qweasf45-1d64-79d9-312-a5dc55fd9dc9/config
```

Depending on the size of your storage backend source, it might take a while to finish. Once it is finished, the pods is going to be restarted.
Once the pod is restarted, you will see again the error saying that vault already has a state. To get rid of that, edit the statefulset and put the original commands/args back:

```
      containers:
      - args:
        - "cp /vault/config/extraconfig-from-values.hcl /tmp/storageconfig.hcl;\n[
          -n \"${HOST_IP}\" ] && sed -Ei \"s|HOST_IP|${HOST_IP?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${POD_IP}\" ] && sed -Ei \"s|POD_IP|${POD_IP?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${HOSTNAME}\" ] && sed -Ei \"s|HOSTNAME|${HOSTNAME?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${API_ADDR}\" ] && sed -Ei \"s|API_ADDR|${API_ADDR?}|g\" /tmp/storageconfig.hcl;\n[
          -n \"${TRANSIT_ADDR}\" ] && sed -Ei \"s|TRANSIT_ADDR|${TRANSIT_ADDR?}|g\"
          /tmp/storageconfig.hcl;\n[ -n \"${RAFT_ADDR}\" ] && sed -Ei \"s|RAFT_ADDR|${RAFT_ADDR?}|g\"
          /tmp/storageconfig.hcl;\n/usr/local/bin/docker-entrypoint.sh vault server
          -config=/tmp/storageconfig.hcl \n"
        command:
        - /bin/sh
        - -ec
```

Recreate the pod, and run a vault status againt it. Vault is going to be initialized already and you will have to unseal it using your old unsealing keys:

`vault operator unseal -migrate`

That's all you need to perform seal and storage migration using vault with helm and Kubernetes.

Last by not least, once the migration is finished and everything is up and running, make sure you enable and configure TLS.

Happy Vault!