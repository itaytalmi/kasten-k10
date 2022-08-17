# Kasten Deployment

- [Kasten Deployment](#kasten-deployment)
  - [Prerequisites](#prerequisites)
  - [Deploy Kasten](#deploy-kasten)
  - [Configure Kasten](#configure-kasten)
  - [Deploy a Sample Application and Perform Backup and Restore](#deploy-a-sample-application-and-perform-backup-and-restore)
  - [Cleanup](#cleanup)
  - [Reference](#reference)

## Prerequisites

- Configure the required permissions for Kasten on vSphere. Use [this Ansible Playbook](https://gitlab.com/skywiz-io/vmware-tanzu/-/tree/master/tkgm/helpers/vsphere-permissions) to do so.
- Allocate an NFS export for storing Kasten backups.

>For testing and lab purposes, you can setup a simple NFS server on Ubuntu using the following commands:
>
>```bash
>sudo apt update
>sudo apt install -y nfs-kernel-server nfs-common portmap
>sudo systemctl start nfs-server
>sudo systemctl enable nfs-server
>sudo mkdir -p /export
>sudo chmod -R 777 /export # Not advised in production as it is not secure
>sudo bash -c "echo '/export  *(rw,sync,no_subtree_check,no_root_squash,insecure)' >> /etc/exports"
>sudo exportfs -a
>showmount -e
>```

## Deploy Kasten

Switch to the appropriate kubectl context.

Create the `kasten-io` namespace.

```bash
kubectl apply -f k10-namespace.yaml
```

Execute the pre-flight checks script.

```bash
curl https://docs.kasten.io/tools/k10_primer.sh | bash
```

Cleanup temporary pre-flight manifest.

```bash
rm -f k10primer.yaml
```

Modify the `k10-ingress-tls-secret.yaml` and `k10-ca-cert.yaml` using your own certificates, then apply both.

```bash
kubectl apply -f k10-ingress-tls-secret.yaml -f k10-ca-cert.yaml
```

Modify the `values.yaml` file using your own configuration, then install the Helm Chart.

``` bash
helm repo add kasten https://charts.kasten.io
helm repo update
helm upgrade -i k10 kasten/k10 -n kasten-io -f values.yaml
```

Make sure the pods are running.

```bash
kubectl get pod -n kasten-io
```

## Configure Kasten

Modify the `k10-nfs.yaml` manifest using your NFS server configuration, then apply it.

```bash
kubectl apply -f k10-nfs.yaml
```

Create NFS profile.

```bash
kubectl apply -f k10-nfs-location-profile.yaml
```

Modify the `k10-infra-vsphere.yaml` manifest using your vSphere configuration, then apply it.

```bash
kubectl apply -f k10-infra-vsphere.yaml
```

>If you navigate to K10 web UI > `Settings` > `Locations` and `Infrastructure`, you can see the resources we have just created declaratively. You can also create any of these resources via the web UI. However, the approach should always be declarative whenever possible.

## Deploy a Sample Application and Perform Backup and Restore

Deploy a sample Wordpress application.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm upgrade -i demo-k10-wordpress -n demo-k10-wordpress --create-namespace bitnami/wordpress \
--set wordpressUsername="demo-k10" \
--set wordpressPassword="Kasten1!" \
--set wordpressBlogName="K10 Demo Blog!"
```

You can run the `kubectl get pvc -n demo-k10-wordpress` command to make sure the PVCs have been created.

For example:

```text
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
data-demo-k10-wordpress-mariadb-0   Bound    pvc-a542e745-3967-42ef-b3fa-baa86aea0495   8Gi        RWO            k8s-storage-policy-vsan   5s
demo-k10-wordpress                  Bound    pvc-b5bd4719-b1b2-4d29-9851-69be6ae2b06b   10Gi       RWO            k8s-storage-policy-vsan   5s
```

Get the service load balancer IP address.

```bash
kubectl get svc -n demo-k10-wordpress demo-k10-wordpress -o jsonpath='{.status.loadBalancer.ingress[].ip}'
```

For example:

```text
10.100.154.12
```

Make sure the application pods are running.

```bash
kubectl get pod -n demo-k10-wordpress
```

For example:

```text
NAME                                  READY   STATUS    RESTARTS   AGE
demo-k10-wordpress-5c558fc667-7r2wr   1/1     Running   0          106s
demo-k10-wordpress-mariadb-0          1/1     Running   0          106s
```

From a web browser, browse to the application. You should see your `K10 Demo Blog` web page.

You can also browse to `http://$LB_ADDRESS/admin` and create a sample blog post for demo purposes.

Create a backup of the Wordpress application via K10 web UI.

From the K10 web UI, navigate to `Applications` and select `Create a Policy` for the `demo-k10-wordpress` application. Select the `Enable Backups via Snapshot Exports`.

>You can click the `YAML` butto at the bottom to create the policy declaratively using kubectl.

Click `Create Policy`.

Select `Run Once` to run the backup and monitor the backup process from the main dashboard.

Wait for the backup to complete.

Once the backup completes, delete the application and its data, so we can test the restore process as well.

```bash
helm uninstall demo-k10-wordpress -n demo-k10-wordpress
kubectl delete ns demo-k10-wordpress
```

To restore the application from the K10 web UI, navigate to `Applications`, click `Filter by Status`, then select `Removed`.
Then click `Restore`, for the `demo-k10-wordpress` application.

Select the backup we created previously, then selected the exported backup under it.

Finally, click `Restore` and monitor the restore process from the main dashboard.

Wait for the restore to complete.

In the meantime, you can inspect the resources being restored on the Kubernetes cluster. For example:

```bash
kubectl get pod -n demo-k10-wordpress
```

Example output:

```text
NAME                                  READY   STATUS              RESTARTS   AGE
demo-k10-wordpress-5c558fc667-p7gr5   0/1     Running             0          7s
demo-k10-wordpress-mariadb-0          0/1     ContainerCreating   0          7s
restore-data-xmz75                    1/1     Terminating         0          50s
```

Get the service load balancer IP address.

```bash
kubectl get svc -n demo-k10-wordpress demo-k10-wordpress -o jsonpath='{.status.loadBalancer.ingress[].ip}'
```

If you browse to the application via a browser, you should see the Wordpress blog, including the sample post you created previously. This indicates that all of your data has been restored successfully.

## Cleanup

To cleanup the sample application and referenced resources, run the following:

```bash
helm uninstall demo-k10-wordpress -n demo-k10-wordpress
kubectl delete ns demo-k10-wordpress
```

## Reference

- [Kasten Docs - vSphere Integration](https://docs.kasten.io/latest/install/storage.html#vsphere)
- [Kasten Docs - Profiles](https://docs.kasten.io/latest/api/profiles.html)
- [Kasten Docs - Install Root CA in K10's namespace](https://docs.kasten.io/latest/install/advanced.html#install-root-ca-in-k10-s-namespace)
- [Kasten Docs - Active Directory Authentication](https://docs.kasten.io/latest/access/authentication.html#active-directory-authentication)
- [Kasten Docs - Restore Applications](https://docs.kasten.io/latest/usage/restore.html)
- [Kasten Docs - Monitoring](https://docs.kasten.io/latest/operating/monitoring.html)
- [Kubernetes Docs - Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes)
