# How to install Velero in an OpenShift environment

## Introduction

So in this lab, The main purpose is to display steps required to install MinIO and Velero in an OpenShift environment and perform a demo backup.

## Clone the Velero repository

The first step is to clone the Velero repository:

` git clone https://github.com/vmware-tanzu/velero.git`

`cd velero`

Before starting, please modify the namespace *examples/minio/00-minio-deployment.yaml* to a more specific namespace.

View the content within the object:

`cat examples/minio/00-minio-deployment.yaml `

## Install MinIO

Now that we cloned the repository, deploying MinIO to your OpenShift environment is very simple:

`oc apply -f examples/minio/00-minio-deployment.yaml`

And you should see the output like this:

>>
    Eduardos-MBP:velero edu$ oc apply -f examples/minio/00-minio-deployment.yaml
    namespace/velero configured
    deployment.apps/minio unchanged
    service/minio unchanged
    job.batch/minio-setup unchanged

## Expose MinIO

Now, we need to expose the MinIO service outside the cluster, so that the velero CLI can interact with it. In OpenShift, this is pretty simple:

`oc project velero`

`oc expose svc minio`

You should see the following output:

>>
    Eduardos-MBP:minio edu$ oc expose svc minio
    route.route.openshift.io/minio exposed

You can get the information about the route by running the following command:

`oc get route minio`

You should be able to open the URL listed there and log in using user: *minio* and password: *minio123*

## Install Velero CLI

Next, we need to install the Velero CLI in the local machine. For Mac, run the following command:

`brew install velero`

## Create credential file

Next, we need to create the Minio credential file. So create a file named *credentials-velero* with the following content:

`[default]`

`aws_access_key_id = minio`

`aws_secret_access_key = minio123`

## Install Velero

We are finally ready to install Velero. Run the following command:

`velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.0 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000`

And you should see the following output:

>>
    edu@eduardos-air velero % velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.0.0 \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
    CustomResourceDefinition/backups.velero.io: attempting to create resource
    CustomResourceDefinition/backups.velero.io: created
    CustomResourceDefinition/backupstoragelocations.velero.io: attempting to create resource
    CustomResourceDefinition/backupstoragelocations.velero.io: created
    CustomResourceDefinition/deletebackuprequests.velero.io: attempting to create resource
    CustomResourceDefinition/deletebackuprequests.velero.io: created
    CustomResourceDefinition/downloadrequests.velero.io: attempting to create resource
    CustomResourceDefinition/downloadrequests.velero.io: created
    CustomResourceDefinition/podvolumebackups.velero.io: attempting to create resource
    CustomResourceDefinition/podvolumebackups.velero.io: created
    CustomResourceDefinition/podvolumerestores.velero.io: attempting to create resource
    CustomResourceDefinition/podvolumerestores.velero.io: created
    CustomResourceDefinition/resticrepositories.velero.io: attempting to create resource
    CustomResourceDefinition/resticrepositories.velero.io: created
    CustomResourceDefinition/restores.velero.io: attempting to create resource
    CustomResourceDefinition/restores.velero.io: created
    CustomResourceDefinition/schedules.velero.io: attempting to create resource
    CustomResourceDefinition/schedules.velero.io: created
    CustomResourceDefinition/serverstatusrequests.velero.io: attempting to create resource
    CustomResourceDefinition/serverstatusrequests.velero.io: created
    CustomResourceDefinition/volumesnapshotlocations.velero.io: attempting to create resource
    CustomResourceDefinition/volumesnapshotlocations.velero.io: created
    Waiting for resources to be ready in cluster...
    Namespace/velero: attempting to create resource
    Namespace/velero: already exists, proceeding
    Namespace/velero: created
    ClusterRoleBinding/velero: attempting to create resource
    ClusterRoleBinding/velero: created
    ServiceAccount/velero: attempting to create resource
    ServiceAccount/velero: created
    Secret/cloud-credentials: attempting to create resource
    Secret/cloud-credentials: created
    BackupStorageLocation/default: attempting to create resource
    BackupStorageLocation/default: created
    Deployment/velero: attempting to create resource
    Deployment/velero: created
    Velero is installed! ⛵ Use 'kubectl logs deployment/velero -n velero' to view the status.

Now, we need to test it.

## Create some Kubernetes resources

To test the backup procedure, let’s first create some Kubernetes resources. It doesn’t matter which resource, so I will stick to a simple one: ConfigMap. Run the following script to create an OpenShift project and many ConfigMaps:

`oc new-project test-backup-your-uniq-id`

`for i in {1..20}; do echo Creating ConfigMap $i; oc create configmap cm-$i --from-literal="key=$i"; done`

The lines above create a new project and 20 ConfigMaps. Run the following command to confirm:

`oc get configmap`

## Back up OpenShift

Now, we can test the backup procedure. For simplicity, we are going to back up only the test-backup project, but the same concept applies to any (or all) projects. Run the following command:

`velero backup create my-backup --include-namespaces test-backup`

and you will see the following output:

>>
    Eduardos-MBP:velero edu$ velero backup create my-backup --include-namespaces test-backup
    Backup request "my-backup" submitted successfully.
    Run `velero backup describe my-backup` or `velero backup logs my-backup` for more details.

After a few seconds, you can check the backup has completed:

`velero backup describe my-backup`

## Simulating loss

Now, let’s simulate some loss:

`oc delete configmap cm-{1..20}`

You can validate by running the following command:

`oc get configmap`

It should return no configmap:

>>
    Eduardos-MBP:velero edu$ oc get configmap
    No resources found.

## Restoring the environment

Let’s now restore the backup that Velero created:

`velero restore create --from-backup my-backup`

Now, we can check the result:

`oc get cm`

And you should see the 20 ConfigMaps restored. Awesome!

## Conclusion

In this lab, We learned how to install and use Velero to back up and restore an OpenShift environment.

Velero makes really simple to back up the etcd data and the Persistent Volumes to an S3 Bucket.
