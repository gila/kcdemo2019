# About

MayaStor is cloud native CSI storage plugin for Kubernetes. It is capable
of dynamic volume provisioning on preconfigured storage pools. The plugin
consists from two parts:

* controller part responsible for volume provisioning called *moac* and
* storage backend part holding provisioned volumes *mayastor*

# Requirements

 * Only works on march icore7 and higher
 * Only tested on k8s 1.14, using on Ubuntu
 * MayaStor DaemonSet  (MDS) requires privileged mode
 * MDS requires 2MB hugepages
 * MDS requires for the purposes of demo the Network Block Device driver (NBD)


# Quick start

We will set up MayaStor CSI plugin on a node with configured kubectl for your
cluster. Following steps were tested with k8s *1.14* version.

1.  Create namespace holding mayastor resources:
    ```bash
    cd deploy
    kubectl create -f namespace.yaml
    ```

2.  Deploy MayaStor CSI plugin:
    ```bash
    kubectl create -f moac-deployment.yaml
    kubectl create -f mayastor-daemonset.yaml
    ```
    Check that moac is running:
    ```bash
    kubectl -n mayastor get pod
    ```
    ```
    NAME                   READY   STATUS    RESTARTS   AGE
    moac-5f7cb764d-sshvz   3/3     Running   0          34s
    ```

3.  Prepare the storage nodes which you would like to use for volume
    provisioning. Each storage node needs at least 512MB of mem in hugepages:
    ```bash
    echo 512 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
    ```
    If you want, you can make this change persistent across reboots by adding following line
    to `/etc/sysctl.conf`:
    ```
    vm.nr_hugepages = 512
    ```

    After adding the hugepages you *must* restart the kubelet. You can verify that
    hugepages haven been created using:

    ```
    cat /proc/meminfo | grep HugePages
    AnonHugePages:         0 kB
    ShmemHugePages:        0 kB
    HugePages_Total:    1024
    HugePages_Free:      671
    HugePages_Rsvd:        0
    HugePages_Surp:        0

    ```

    And load nbd resp. xfs kernel modules which is needed for publishing
    volumes resp. mounting filesystems.
    ```
    modprobe nbd
    modprobe xfs
    ```
    And, if you want, make this change persistent across reboots by adding lines with
    `nbd` and `xfs` to `/etc/modules-load.d/modules.conf`.

4.  Label the storage nodes (here we use node "node1"):
    ```bash
    kubectl label node node1 openebs.io/engine=mayastor
    ```
    Check that mayastor has been started on the storage node:
    ```bash
    kubectl -n mayastor get pod
    ```
    ```
    NAME                   READY   STATUS    RESTARTS   AGE
    mayastor-gldv8         3/3     Running   0          81s
    moac-5f7cb764d-sshvz   3/3     Running   0          6h46m
    ```

5.  Create a storage pool for volume provisioning (replace `disks` and `node`
    values as appropriate):
    ```bash
    cat <<EOF | kubectl create -f -
    apiVersion: "openebs.io/v1alpha1"
    kind: MayastorPool
    metadata:
      name: pool
    spec:
      node: node1
      disks: ["/dev/vdb"]
    EOF
    ```
    Check that the pool has been created (Note that the `State` *must be* `ONLINE`):
    ```bash
    kubectl describe msp pool
    ```
    ```
    Name:         pool
    Namespace:
    Labels:       <none>
    Annotations:  <none>
    API Version:  openebs.io/v1alpha1
    Kind:         MayastorPool
    Metadata:
      Creation Timestamp:  2019-04-09T21:41:47Z
      Generation:          1
      Resource Version:    1281064
      Self Link:           /apis/openebs.io/v1alpha1/mayastorpools/pool
      UID:                 46aa02bf-5b10-11e9-9825-589cfc0d76a7
    Spec:
      Disks:
        /dev/vdb
      Node:  node1
    Status:
      Capacity:  10724835328
      Reason:
      State:     ONLINE
      Used:      0
    Events:      <none>
    ```

6.  Create storage class using mayastor CSI plugin for volume provisioning:
    ```bash
    cat <<EOF | kubectl create -f -
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: mayastor
    provisioner: io.openebs.csi-mayastor
    EOF
    ```

7.  Create persistent volume claim (PVC):
    ```bash
    cat <<EOF | kubectl create -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: ms-volume-claim
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: mayastor
    EOF
    ```
    Verify that the PVC and persistent volume (PV) for the PVC have been
    created:
    ```bash
    kubectl get pvc
    ```
    ```
    NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    ms-volume-claim   Bound    pvc-21d56e09-5b78-11e9-905a-589cfc0d76a7   1Gi        RWO            mayastor       22s
    ```
    ```bash
    kubectl get pv
    ```
    ```
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
    pvc-21d56e09-5b78-11e9-905a-589cfc0d76a7   1Gi        RWO            Delete           Bound    default/ms-volume-claim   mayastor                27s
    ```

8.  Deploy a pod with fio tool which will be using the PV:
    ```bash
    cat <<EOF | kubectl create -f -
    kind: Pod
    apiVersion: v1
    metadata:
      name: fio
    spec:
      volumes:
        - name: ms-volume
          persistentVolumeClaim:
           claimName: ms-volume-claim
      containers:
        - name: fio
          image: dmonakhov/alpine-fio
          args:
            - sleep
            - "1000000"
          volumeMounts:
            - mountPath: "/volume"
              name: ms-volume
    EOF
    ```
    Check that it has been successfully deployed:
    ```
    kubectl get pod
    ```

9.  Run fio on the volume for 30s:
    ```bash
    kubectl exec -it fio -- fio --name=benchtest --size=800m --filename=/volume/test --direct=1 --rw=randrw --ioengine=libaio --bs=4k --iodepth=16 --numjobs=1 --time_based --runtime=60
    ```

# QuickStart microk8s

1. Install microk8s 1.14 using:

```
snap install microk8s --classic --channel=1.14/stable
```

2. Update the configuration of microk8s to allow for privileged mode:

```
# kubelet config
# - add `--allow-privileged=true`
systemctl restart snap.microk8s.daemon-kubelet.service

#kube-apiserver config
# - add `--allow-privileged=true`
systemctl restart snap.microk8s.daemon-apiserver.service
```

3. Verify the changes where successfully applied:

```
~snap services
Service                             Startup  Current  Notes
microk8s.daemon-apiserver           enabled  active   -
microk8s.daemon-apiserver-kicker    enabled  active   -
microk8s.daemon-containerd          enabled  active   -
microk8s.daemon-controller-manager  enabled  active   -
microk8s.daemon-etcd                enabled  active   -
microk8s.daemon-kubelet             enabled  active   -
microk8s.daemon-proxy               enabled  active   -
microk8s.daemon-scheduler           enabled  active   -
```

4. If requirerd make sure forwarding rules are enabled:

```
sudo iptables -P FORWARD ACCEPT
```

5. Make *sure* you update the arguments in the YAML file to point to the proper microk8s directories. For example: 
```
      - name: registration-dir
        hostPath:
          path: /var/snap/microk8s/common/var/lib/kubelet/plugins_registry/
          type: Directory
```

Notice how the path now, is prepended with `/var/snap/microk8s/common/` compared to the orignal file. Depending on what distro you are using, this might differ. Then lastly, follow the steps from the regular Quickstart.


# Know issues

* The MayaStor service suddenly restarts when mounting a PVC, with exit code `132`

 This is due to a SIGILL, which means the container is has not been
 compiled properly from our CI system for your CPU architecture. As a result we
 ask the CPU something that it does not know how to do.

* Missing finalizers

    Finalizers have not been implemented yet, there for, it is important to follow
    the proper order of tear down:

     - delete the pods using a mayastor PVC
     - delete the PVC
     - delete the MSP
     - delete the DaemonSet






