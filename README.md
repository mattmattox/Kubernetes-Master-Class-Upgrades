# Kubernetes-Master-Class-Upgrades
Kubernetes Master Class: A Seamless Approach to Rancher &amp; Kubernetes Upgrades

## Terms
- **Rancher Server** is a set of pods that run the main orchestration engine and UI for Rancher.
- **RKE** (Rancher Kubernetes Engine) is the tool Rancher uses to create and manage Kubernetes clusters
- **Local/upstream cluster** This is the cluster where the Rancher server is installed, this is usually an RKE built cluster)
- **Downstream cluster(s)** are Kubernetes cluster that Rancher is managing

## High-Level rules
The following are the high-level rules for planning a Rancher/Kubernetes/Docker upgrade.
- Do not rush an upgrade.
- Do not stack upgrades (We recommended at least 24hours between upgrades)
- Make sure you have a good backup
- The recommended order of upgrades is Rancher, Kubernetes, and then Docker.
- All upgrades should be tested in a lab or non-prod environment before being deployed to Production.
- Review all release notes [link](https://github.com/rancher/rancher/releases/tag/v2.5.5)
- Review the support matrix [link](https://rancher.com/support-maintenance-terms/all-supported-versions/rancher-v2.5.5/)
- It is not required, but we recommended pausing any CI/CD pipelines using the Rancher API during an upgrade.

## Picking a version
Please see the following recommendations when planning version upgrades.
- **Rancher**: perform one minor version jump at a time
For example: when upgrading from v2.1.x -> v2.3.x. We encourage upgrading v2.1.x -> v2.2.x -> v2.3.x but this is not required.
- **Kubernetes**: perform no more than 2 minor versions at a time, ideally avoid skipping minor versions entirely as this can increase the chances of an issue due to accumulated changes
For example: when upgrading from v1.13.x -> v1.19.x we encourage upgrading v1.13.x -> v1.15.x -> v1.17.x -> v1.19.x
- **RKE**: perform one major RKE versions jump at a time
For example: when upgrading from v0.1.x -> v1.1.0 instead do v0.1.x -> v0.2.x -> v.0.3.x -> v1.0.x -> v1.1.x


## Creating your change control

### Scheduled change window
- **Rancher upgrade** - 30Mins for install with 30mins for rollback
- **Kubernetes upgrade** - 60Mins for install which may be longer for larger clusters with 60Mins for troubleshooting/rollback

### Effect / Impact during the change window
- **Rancher upgrade** - Only management of Rancher and downstream clusters are impacted, Applications shouldn't know that anything is being done. But any CI/CD pipelines should be paused
- **Kubernetes upgrade of the local cluster** - The Rancher UI should disconnect and reconnect after a few mins due to the ingress-controllers being restarted.
- **Kubernetes upgrade of downstream clusters** - Applications might see a short network blip as ingress-controllers and networking is restarted. See [link](https://rancher.com/blog/2020/zero-downtime/) for more details

### Maintenance window
- **Rancher upgrade** - A maintenance window is not required, but CI/CD pipelines should be paused.
- **Kubernetes upgrade of the local cluster** - A maintenance window is not needed, but CI/CD pipelines should be paused.
- **Kubernetes upgrade of downstream clusters** - This should be done during a maintenance window or a quiet time

## Rancher Upgrade – Prep work
- Check if the Rancher UI is accessible
    - Check if all clusters in UI are in Active state
    - Check if all pods in kube-system and cattle-system namespaces are running in both the local and downstream clusters.
        ```
        kubectl get pods -n kube-system
        kubectl get pods -n cattle-system
        ```
- Verify etcd has scheduled snapshots configured, and these are working.
    - **RKE**: if Rancher is deployed on a Kubernetes cluster built with RKE, verify etcd snapshots are enabled and working, on etcd nodes you can confirm with the following:
        ```
        ls -l /opt/rke/etcd-snapshots
        docker logs etcd-rolling-snapshots
        ```
    - **k3s**: if Rancher is deployed on a k3s Kubernetes cluster, ensure scheduled backups are configured and working. Please see the k3s [documentation](https://rancher.com/docs/k3s/latest/en/) pages for further information on this.
- Create a one-time datastore snapshot, please see the following documentation for RKE and k3s, and the single node Docker install options for more information
    - **RKE**: check for expired/expiring Kubernetes certs
        ```
        for i in $(ls /etc/kubernetes/ssl/*.pem|grep -v key); do echo -n $i" "; openssl x509 -startdate -enddate -noout -in $i | grep 'notAfter='; done
        ```

## Rancher Upgrade - Change
- Update helm repo cache
    ```
    helm repo update
    helm fetch rancher-stable/rancher
    ```
- Verify you’re connected to the correct cluster
    ```
    kubectl get nodes -o wide
    ```
- Take an etcd snapshot
    ```
    rke etcd snapshot-save --config cluster.yaml --name pre-rancher-upgrade-`date '+%Y%m%d%H%M%S'`
    ```    
- Grab the current helm values using helm get values rancher -n cattle-system
    Example output:
    ```
    USER-SUPPLIED VALUES:
    antiAffinity: required
    auditLog:
      level: 2
    hostname: rancher.example.com
    ingress:
      tls:
        source: secret
    ```
- Use the values to build your upgrade command
    **NOTE**: The only thing you should change is the version flag.
    ```
    helm upgrade --install rancher rancher-stable/rancher \
    --namespace cattle-system \
    --set hostname=rancher.example.com \
    --set ingress.tls.source=secret \
    --set auditLog.level=2 \
    --set antiAffinity=required \
    --version 2.5.5
    ```
- Wait for the upgrade to finish
    ```
    kubectl -n cattle-system rollout status deploy/rancher
    ```
- Official Rancher upgrade [documentation](https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/upgrades/)

## Rancher Upgrade – Verify
- Check if the Rancher UI is accessible
    - Check if all clusters in UI are in an Active state
    - Check if all pods in kube-system and cattle-system namespaces are running in both the local and downstream clusters.
        ```
        kubectl get pods -n kube-system
        kubectl get pods -n cattle-system
        ```
- Verify new Rancher version (Bottom Left corner)
- Verify all Rancher, cattle-cluster-agent, and cattle-node-agent is running on the new version on the local cluster
    ```
    kubectl get pods -n cattle-system -o wide
    ```
- Verify all downstream cluster are Active
- Verify all Rancher, cattle-cluster-agent, and cattle-node-agent runs on the new version on all downstream clusters.
    ```
    kubectl get pods -n cattle-system -o wide
    ```
Take a post-upgrade etcd snapshot
    ```
    rke etcd snapshot-save --config cluster.yaml --name post-rancher-upgrade-`date '+%Y%m%d%H%M%S'`
    ```

## Rancher Upgrade – Backout
- You can not downgrade Rancher; you must do an etcd restore [Documentation](https://rancher.com/docs/rke/latest/en/etcd-snapshots/restoring-from-backup/)
    ```
    rke etcd snapshot-restore --name pre-rancher-upgrade-..... --config ./cluster.yaml
    ```

## RKE Upgrade – Prep work
- Verify the correct `cluster.yaml` and `cluster.rkestate` file
- Verify SSH access to all nodes in the cluster
- Verify all nodes are Ready
    ```
    kubectl get nodes -o wide
    ```
- Verify all pods are Healthy
    ```
    kubectl get pods --all-namespaces -o wide | grep -v 'Running\|Completed'
    ```
    - We're looking for Pods crashing or stuck.
- Verify Kubernetes version is available in RKE
    ```
    rke config --list-version --all –print
    ```
    - You might need to upgrade to a newer RKE version if the recommend k8s version isn't available.

## RKE Upgrade – Change
- Take an etcd snapshot
    ```
    rke etcd snapshot-save --config cluster.yaml --name pre-k8s-upgrade-`date '+%Y%m%d%H%M%S'`
    ```
- Change `kubernetes_version` in the `cluster.yaml`
    ```
    kubernetes_version: "1.19.7-rancher1-1"
    ```
- If you have an air-gapped setup, please see [documentation](https://rancher.com/docs/rke/latest/en/config-options/system-images/)

## RKE Upgrade - Verify
- Verify all nodes are Ready and at the new version
    ```kubectl get nodes -o wide
    ```
- Verify all pods are Healthy
    ```
    kubectl get pods --all-namespaces -o wide | grep -v 'Running\|Completed’
    ```
    - All pods should be healthy; we're looking for Pods crashing or stuck.

## RKE Upgrade – Backout
- You can not downgrade Rancher; **you must do an etcd restore**
    ```
    rke etcd snapshot-restore --name pre-k8s-upgrade-..... --config ./cluster.yaml
    ```
    - [Documentation](https://rancher.com/docs/rke/latest/en/etcd-snapshots/restoring-from-backup/)

## Common issues

### Missing `cluster.yaml` and `cluster.rkestate`

#### Setting up lab environment
- Build a standard RKE cluster [Documentation](https://rancher.com/docs/rke/latest/en/installation/#deploying-kubernetes-with-rke)
- Delete `cluster.rkestate`
- Delete `kube_config_cluster.yml`

#### Reproducing the issue
- `rke up`
- You should see rke generating new certificates (See example output below)
```
INFO[0004] [certificates] Generating CA kubernetes certificates
INFO[0005] [certificates] Generating Kubernetes API server aggregation layer requestheader client CA certificates
INFO[0005] [certificates] GenerateServingCertificate is disabled, checking if there are unused kubelet certificates
INFO[0005] [certificates] Generating Kubernetes API server certificates
INFO[0006] [certificates] Generating Service account token key
INFO[0006] [certificates] Generating Kube Controller certificates
INFO[0006] [certificates] Generating Kube Scheduler certificates
INFO[0006] [certificates] Generating Kube Proxy certificates
INFO[0007] [certificates] Generating Node certificate
INFO[0007] [certificates] Generating admin certificates and kubeconfig
```

#### Resolution

- SSH to one of controlplane nodes
- Run the [script](https://raw.githubusercontent.com/rancherlabs/support-tools/master/how-to-retrieve-kubeconfig-from-custom-cluster/rke-node-kubeconfig.sh) and follow the instructions given to get a kubeconfig file for the cluster.
- Run the [script](https://raw.githubusercontent.com/rancherlabs/support-tools/master/how-to-retrieve-cluster-yaml-from-custom-cluster/cluster-yaml-recovery.sh) and follow the instructions given to get a cluster.yaml and cluster.rkestate file for the cluster.
- Copy the files cluster.yml, cluster.rkestate, and kube_config_cluster.yml to safe location.
