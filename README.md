Enabling GPU workloads with containerd as CRI in a Kubernetes cluster

<b>Generating [nvidia_gpu_operator.yaml.orig](nvidia_gpu_operator.yaml.orig) </b>


a.  Official documentation of NVIDIA GPU Operator on containerd available at https://developer.nvidia.com/blog/announcing-containerd-support-for-the-nvidia-gpu-operator/


 Specifically, nvidia/gpu-operator can be installed via Helm.
```
helm install --wait --generate-name \
   nvidia/gpu-operator \
   --set operator.defaultRuntime="containerd"
```

b. Use `helm template` to convert the helm chart to raw Kubernetes manifests.

```
$ helm template nvidia nvidia/gpu-operator --set operator.defaultRuntime="containerd" > nvidia_gpu_operator.yaml
```

 <b>Customize the above and generate [nvidia_gpu_operator.yaml](nvidia_gpu_operator.yaml)</b>

 a. Add nodeSelector and tolerations to the Daemonsets, so that the pods are scheduled on the appropriate P instance nodes.

 b. Disable nvidia driver install, since our AMIs pre-bake the drivers.

 c. Apply this customized yaml using `k apply -f nvidia_gpu_operator.yaml`


 <b>Errors observed</b>

 1. On the GPU instances, Kubelet service dies. As a result, the nvidia GPU daemonset pods are stuck in pending, because the nodes are in `NotReady` state.

```
# systemctl status -l kubelet.service
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubelet-args.conf, 30-kubelet-extra-args.conf
   Active: inactive (dead) since Wed 2022-03-23 00:33:55 UTC; 57min ago
     Docs: https://github.com/kubernetes/kubernetes
  Process: 334665 ExecStart=/usr/bin/kubelet --cloud-provider aws --config /etc/kubernetes/kubelet/kubelet-config.json --kubeconfig /var/lib/kubelet/kubeconfig --container-runtime docker --resolv-conf /etc/resolv.conf.save --network-plugin cni $KUBELET_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=0/SUCCESS)
  Process: 334651 ExecStartPre=/sbin/iptables -P FORWARD ACCEPT -w 5 (code=exited, status=0/SUCCESS)
 Main PID: 334665 (code=exited, status=0/SUCCESS)

Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: I0323 00:33:55.368920  334665 kubelet.go:1956] SyncLoop (PLEG): "nvidia-node-feature-discovery-worker-hh9vz_addon-gpuoperator-ns(587af86e-38ea-473f-9e3f-4df4e48b2882)", event: &pleg.PodLifecycleEvent{ID:"587af86e-38ea-473f-9e3f-4df4e48b2882", Type:"ContainerStarted", Data:"ab45c6b8cef8010c53e9d9929a4a6f567a6dc140dfd584229b6e1440860f1b2d"}
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: I0323 00:33:55.371036  334665 kubelet.go:1956] SyncLoop (PLEG): "dns-healthcheck-workflow-n5nbt-701182422_addon-active-monitor-ns(9f66162f-da47-4b49-8ce6-479c558b44ba)", event: &pleg.PodLifecycleEvent{ID:"9f66162f-da47-4b49-8ce6-479c558b44ba", Type:"ContainerStarted", Data:"26f747e925ac8d8f7347506b9d4624b39142c0c61d23ba916c842b0f88e5d224"}
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: I0323 00:33:55.371102  334665 kubelet.go:1956] SyncLoop (PLEG): "dns-healthcheck-workflow-n5nbt-701182422_addon-active-monitor-ns(9f66162f-da47-4b49-8ce6-479c558b44ba)", event: &pleg.PodLifecycleEvent{ID:"9f66162f-da47-4b49-8ce6-479c558b44ba", Type:"ContainerStarted", Data:"cf906426a941ff934561d9bcda971a2e090570d21a168ebc7be636f23be8a523"}
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: I0323 00:33:55.371222  334665 kubelet.go:1956] SyncLoop (PLEG): "dns-healthcheck-workflow-n5nbt-701182422_addon-active-monitor-ns(9f66162f-da47-4b49-8ce6-479c558b44ba)", event: &pleg.PodLifecycleEvent{ID:"9f66162f-da47-4b49-8ce6-479c558b44ba", Type:"ContainerStarted", Data:"7d0fd012ffe7079d4905a346232e0a61ae27e872ec1ba9e62762f3273e06a835"}
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: WARNING: 2022/03/23 00:33:55 grpc: addrConn.createTransport failed to connect to {unix:///run/containerd/containerd.sock  <nil> 0 <nil>}. Err :connection error: desc = "transport: failed to write client preface: write unix @->/run/containerd/containerd.sock: write: broken pipe". Reconnecting...
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: WARNING: 2022/03/23 00:33:55 grpc: addrConn.createTransport failed to connect to {/run/containerd/containerd.sock  <nil> 0 <nil>}. Err :connection error: desc = "transport: failed to write client preface: write unix @->/run/containerd/containerd.sock: write: broken pipe". Reconnecting...
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: WARNING: 2022/03/23 00:33:55 grpc: addrConn.createTransport failed to connect to {/run/containerd/containerd.sock  <nil> 0 <nil>}. Err :connection error: desc = "transport: failed to write client preface: write unix @->/run/containerd/containerd.sock: write: broken pipe". Reconnecting...
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal systemd[1]: Stopping Kubernetes Kubelet...
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal kubelet[334665]: I0323 00:33:55.546422  334665 dynamic_cafile_content.go:182] Shutting down client-ca-bundle::/etc/kubernetes/pki/ca.crt
Mar 23 00:33:55 ip-10-225-49-119.vpc.internal systemd[1]: Stopped Kubernetes Kubelet.

```

Here's the containerd config, just in case:

```
# cat /etc/containerd/config.toml 
disabled_plugins = ["io.containerd.snapshotter.v1.aufs", "io.containerd.snapshotter.v1.btrfs", "io.containerd.snapshotter.v1.devmapper", "io.containerd.snapshotter.v1.zfs"]
root = "/var/lib/containerd"
state = "/run/containerd"
version = 2

[plugins]

  [plugins."io.containerd.grpc.v1.cri"]
    max_concurrent_downloads = 3

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "nvidia"
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
            BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime"

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-experimental]
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia-experimental.options]
            BinaryName = "/usr/local/nvidia/toolkit/nvidia-container-runtime-experimental"

    [plugins."io.containerd.grpc.v1.cri".registry]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.intuit.com"]
          endpoint = ["https://docker.intuit.com", "https://docker-readonly.intuit.com"]

```


2. I can manually restart the kubelet and the nodes are now ready for scheduling pods.

3. GPU Daemonset pods start running in the GPU instance nodes. However, they crash due to `PodCrashLoopBackOff`


```
$ k get pods
NAME                                                    READY   STATUS                   RESTARTS   AGE
gpu-feature-discovery-vgkcb                             0/1     Init:RunContainerError   3          8m5s
gpu-feature-discovery-vgss6                             0/1     Pending                  0          8m5s
gpu-operator-5ffc545c4b-m2rzs                           1/1     Running                  0          8m24s
nvidia-container-toolkit-daemonset-7n86l                0/1     Pending                  0          8m6s
nvidia-container-toolkit-daemonset-tqxkk                1/1     Running                  1          8m6s
nvidia-dcgm-exporter-5zt6w                              0/1     Pending                  0          8m5s
nvidia-dcgm-exporter-md7x6                              0/1     Init:RunContainerError   3          8m5s
nvidia-device-plugin-daemonset-v7mf2                    0/1     Init:RunContainerError   3          8m5s
nvidia-device-plugin-daemonset-wx2tl                    0/1     Pending                  0          8m5s
nvidia-node-feature-discovery-master-6f6f867457-bx27f   1/1     Running                  0          8m25s
nvidia-node-feature-discovery-worker-ptchw              0/1     Pending                  0          8m25s
nvidia-node-feature-discovery-worker-s5894              1/1     Running                  0          8m25s
nvidia-operator-validator-hkjh6                         0/1     Init:CrashLoopBackOff    2          8m5s
nvidia-operator-validator-m82xq                         0/1     Pending                  0          8m5s

```

4. Events from the `Init:RunContainerError` pods

```
$ k describe pod gpu-feature-discovery-vgkcb (same errors from other pods: nvidia-dcgm-exporter-md7x6, nvidia-device-plugin-daemonset-v7mf2 and nvidia-operator-validator-hkjh6)

...
..
...
...

Events:
  Type     Reason                  Age                From               Message
  ----     ------                  ----               ----               -------
  Normal   Scheduled               8m33s              default-scheduler  Successfully assigned addon-gpuoperator-ns/gpu-feature-discovery-vgkcb to ip-10-225-49-119.us-west-2.compute.internal
  Warning  FailedCreatePodSandBox  5m14s              kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to get sandbox runtime: no runtime for "nvidia" is configured
  Normal   Pulled                  35s (x4 over 75s)  kubelet            Container image "nvcr.io/nvidia/cloud-native/gpu-operator-validator:v1.10.0" already present on machine
  Normal   Created                 35s (x4 over 75s)  kubelet            Created container toolkit-validation
  Warning  Failed                  34s (x4 over 75s)  kubelet            Error: failed to create containerd task: OCI runtime create failed: container_linux.go:367: starting container process caused: process_linux.go:495: container init caused: Running hook #0:: error running hook: exit status 1, stdout: , stderr: /usr/local/nvidia/toolkit/nvidia-container-cli.real: /lib64/libc.so.6: version `GLIBC_2.27' not found (required by /usr/local/nvidia/toolkit/libnvidia-container.so.1): unknown
  Warning  BackOff                 10s (x7 over 73s)  kubelet            Back-off restarting failed container

```




