# kubernetes-nexus

Nexus Repository Manager OSS (3.13.0) on top of Kubernetes.

## Table of Contents

- [Pre-Requisites](#pre-requisites)
- [Deployment](#deployment)
  - [Deploying Nexus](#deploying-nexus)
  - [Securing Nexus with HTTPS](#securing-nexus-with-https)
  - [Configuring Nexus](#configuring-nexus)
  - [Configuring Backup Retention](#configuring-backup-retention)
- [Using Nexus](#usage)
  - [Docker](#usage-docker)
  - [Maven](#usage-maven)
  - [Gradle](#usage-gradle)
  - [sbt](#usage-sbt)
  - [Python](#usage-python)
- [Backup and Restore](#backup-and-restore)
  - [Backup](#backup)
  - [Restore](#restore)

## Pre-Requisites

- A working Kubernetes cluster (v1.10.7 or newer) with Cloud Storage read-write
permissions enabled (`https://www.googleapis.com/auth/devstorage.read_write` scope)
- A working installation of `kubectl` configured to access the
  cluster.
- A working installation of `gcloud` configured to access the Google Cloud
  Platform project.
- A global static IPv4 address (e.g., `static-ip-name`).
- A DNS A record `nexus.example.com` pointing to this IPv4 address.
- A DNS CNAME record `containers.example.com` pointing to
  `nexus.example.com`.
- A [Google Cloud Storage](https://cloud.google.com/storage/) bucket (e.g.,
  `nexus-backup`).


**Attention**: On RBAC-enabled clusters, and due to [the way GKE checks permissions](https://cloud.google.com/container-engine/docs/role-based-access-control#defining_permissions_in_a_role), one must also grant themselves the `cluster-admin` role *manually* before proceeding:

```bash
$ MY_GCLOUD_USER=$(gcloud info | grep Account | awk -F'[][]' '{print $2}')
$ kubectl create clusterrolebinding ${MY_GCLOUD_USER} --clusterrole=cluster-admin --user=${MY_GCLOUD_USER}
```

## Deployment

### Deploying Nexus

The very first thing one **must do** after deploying Nexus is to log-in into the
Nexus UI with the default credentials (`admin:admin123` ) and proceed to change
the password to something more secure. But one **must not** do it right away!
One will do it after [securing Nexus with HTTPS](#securing-nexus-with-https).

The `nexus-backup` container uses the aforementioned credentials to
access the Nexus API and execute backups. The same credentials are provided
during deployment. Therefore, before deploying Nexus, logging-in and changing
the password as instructed above, one **must** decide what password will be set
and create a Kubernetes secret containing it, and only then deploy Nexus.
That can be done as follows:

```bash
$ cd kubernetes/
$ NEXUS_CREDENTIALS=$(echo -n 'admin:<new-password>' | base64)
$ NEXUS_AUTH=$(echo -n "Basic ${NEXUS_CREDENTIALS}" | base64)
$ sed -i.bkp "s/QmFzaWMgWVdSdGFXNDZZV1J0YVc0eE1qTT0=/${NEXUS_AUTH}/" nexus-secret.yaml
```

One must also update the contents of `nexus-statefulset.yaml` and `nexus-ingress.yaml` according to one's setup _before_ proceeding.

After updating `nexus-secret.yaml`, `nexus-statefulset.yaml` and `nexus-ingress.yaml`, one may deploy Nexus as follows:

**Attention**: If one wants to have GCP IAM authentication enabled, one must
follow [these instructions](docs/admin/configuring-nexus-proxy.md) instead.

```bash
$ kubectl create -f nexus-secret.yaml
$ kubectl create -f nexus-statefulset.yaml
$ kubectl create -f nexus-proxy-svc.yaml
$ kubectl create -f nexus-ingress.yaml
```

One should allow for 5 to 10 minutes for GCLB to update. Nexus should then
become available over HTTP at http://nexus.example.com.

### Securing Nexus with HTTPS

In order to secure Nexus external access one must configure HTTPS access.
The easiest and cheapest way to obtain a trusted TLS certificate is using
[Let's Encrypt](https://letsencrypt.org/), and the easiest way to automate the
process of obtaining and renewing certificates from Let's Encrypt is by using
[`cert-manager`](https://github.com/jetstack/cert-manager):

The easiest way is to install `cert-manager` via Helm, but a static manifest is available as well.
One can follow [these](https://cert-manager.readthedocs.io/en/latest/getting-started/2-installing.html) installation instructions.

As soon as it starts, `cert-manager` will start monitoring _Ingress_ resources and
requesting certificates from Let's Encrypt.

After installation, one will need to setup an issuer and actually request a certificate for Nexus:

**Attention:** One must edit the `certificate.yaml` file according to one's setup _before_ running the following commands.

```bash
$ cd ../cert-manager/
$ kubectl create -f issuer.yaml
$ kubectl create -f certificate.yaml
```

**NOTE**: Let's Encrypt must be able to reach port `80` on domains for which
certificates are requested, hence the annotation `kubernetes.io/ingress.allow-http`
in [`nexus-ingress.yaml`](kubernetes/nexus-ingress.yaml) must be set to `"true"`.

If everything goes well, after a while one will
be able to access https://nexus.example.com securely **and** proceed to log-in
into Nexus with the default credentials (`admin:admin123`), and finally change
the `admin` password to the secure password one decided above.

### Configuring Nexus

One should head over to
[`docs/admin/configuring-nexus.md`](docs/admin/configuring-nexus.md)
for information on how to configure Nexus.

### Configuring Backup Retention

**Attention**: As mentioned in the pre-requisites, the GKE cluster needs read-write
permissions on GCP Cloud Storage in order to upload backups.

The backup procedure uses Google Cloud Storage to save backups. In order to
configure a backup retention policy, one should head over to the `backup-lifecycle`
directory and install one of the available policies by running:

```bash
$ ./gsutil-lifecycle-set <policy-file> <bucket-name>
```

Google Cloud Storage will then automatically purge backups older than the number
of days specified.

<a id="usage">

## Using Nexus

Below are linked detailed instructions on how to configure a bunch of tools
in order to download and upload artifacts from and to Nexus.

<a id="usage-docker">

### Docker

Please, read [Using Nexus with Docker](docs/usage/using-nexus-with-docker.md).

<a id="usage-maven">

### Maven

Please, read [Using Nexus with Maven](docs/usage/using-nexus-with-maven.md).

<a id="usage-gradle">

### Gradle

Please, read [Using Nexus with Gradle](docs/usage/using-nexus-with-gradle.md).

<a id="usage-sbt">

### sbt

Please, read [Using Nexus with sbt](docs/usage/using-nexus-with-sbt.md).

<a id="usage-python">

### Python

Please, read [Using Nexus with Python](docs/usage/using-nexus-with-python.md).

## Backup and Restore

**Attention**: As mentioned in the pre-requisites, the GKE cluster needs read-write
permissions on GCP Cloud Storage in order to upload backups.

### Backup

Nexus has a built-in
[_Export databases for backup_](https://help.sonatype.com/repomanager3/backup-and-restore/configure-and-run-the-backup-task) task which can be used to backup configuration and metadata. However, the generated backup doesn't include [blob stores](https://help.sonatype.com/repomanager3/high-availability/configuring-blob-stores), rendering it semi-useless in a disaster recovery scenario. It is thus of the utmost importance to backup blob stores separately and at roughly the same time this task runs in order to achieve consistent backups.

This is the role of the
[`nexus-backup`](https://github.com/travelaudience/docker-nexus-backup)
container — a container with a script which makes backups of the `default`
blob store and then uploads them on a Google Cloud Storage bucket alongside the
data generated by Nexus built-in backup task. These backups are triggered by
a Nexus task of type _Execute script_ (to be configured as described in
[`docs/admin/configuring-nexus.md`](docs/admin/configuring-nexus.md#configure-backup-tasks-and-create-backup-scripts)).
The script reacts to changes made to

```
/nexus-data/backup/.backup
```

and initiates the backup process:

1. Check if Nexus can be reached. Proceed only if Nexus is reached successfully.
1. Retrieve the current timestamp in order to tag the backup appropriately.
1. Make sure that the scripts used to start/stop repositories are installed.
1. Stop all the repositories so that no writes are made during the backup.
1. Give Nexus a _grace period_ for the configuration and metadata backup task to
   complete.
1. Archive the `default` blob store using `tar`,
   [_streaming_](https://cloud.google.com/storage/docs/streaming) the resulting
   file to the target bucket.
1. Archive the backup files generated by Nexus using `tar`, _streaming_ the
   resulting file to the target bucket.
1. Cleanup all leftover `*.bak` files in `/nexus-data/backup`.
1. Start all the repositories so that Nexus can become available again.

It is advisable to configure this backup procedure to run at off-peak hours, as
described in the aforementioned document.

### Restore

In a disaster recovery scenario, the latest backup made by the `nexus-backup`
container should be restored. In order to achieve this, Nexus must be stopped.
The procedure goes as follows:

```
$ kubectl exec -i -t nexus-0 --container nexus -- sh # Enter the container.
$ mv /etc/service/nexus/ /nexus-service/         # Prevent `runsvdir` from respawning Nexus after the next step is performed.
$ pkill java                                     # Ask for Nexus to terminate gracefully.
```

**Attention**: after terminating Nexus, with the default readiness/liveness probe settings in the chart kubernetes will restart the nexus pod before you've been able to untar the back-up files. If you meet that problem (the `sh` into the `nexus` container terminates abruptly with exit code 137), you may overcome it by extending the time limits of the readiness/liveness probes: `kubectl edit statefulset nexus` and change `spec.template.spec.containers.livenessProbe.failureThreshold` and `spec.template.spec.containers.readinessProbe.failureThreshold` for the `nexus` container to a large value, e.g. 100. Once you're done with the restore, don't forget to reset those values to their initial setting!!

At this point, Nexus is stopped but the container is still running, giving one a
chance to perform the restore procedure.

**Attention**: One must not close this terminal window just yet.

One should now open another terminal window and login into the `nexus-backup` container in order to use `gsutil` to retrieve the desired backup.

```shell
$ kubectl exec -i -t nexus-0 --container nexus-backup -- /bin/bash
```

Now one should go as follows:

1. Retrieve from GCS the `blobstore.tar` and `databases.tar` files
   from the last known good backup.
1. Remove everything _under_ `/nexus-data/backup/`,
   `/nexus-data/blobs/default/` and `/nexus-data/db/`
1. Run `tar -xvf blobstore.tar --strip-components 3 -C /nexus-data/blobs/default/`.
1. Run `tar -xvf databases.tar --strip-components 2 -C /nexus-data/restore-from-backup/`.

At this point the backup is ready to be restored by Nexus and one can leave the _nexus-backup_ container.
One must now go back to the terminal of the `nexus` container and run the following commands:

```
$ mv /nexus-service/ /etc/service/nexus/         # Make `runsvdir` start Nexus again.
$ exit                                           # Bye!
```

This will automatically start Nexus and perform the restore process.

If one watches the `nexus` container logs as it boots, one should see an indication
that a restore procedure is in place. After a few minutes access should be
restored with the last known good backup in place.
