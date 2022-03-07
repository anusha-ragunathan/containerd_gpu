# containerd_gpu

For containerd as the CRI, we are trying to enable GPU based instances for our Kubernetes workloads.
We encountered issues in the 'p type GPU instances', where the nvidia kernel module is cannot be unloaded.
This results in the related daemonsets not getting deployed to the GPU nodes and the GPU applications cannot be deployed.


1. Provision the GPU nodes.

$ k get nodes
NAME                                          STATUS   ROLES                                  AGE   VERSION
ip-10-225-48-29.us-west-2.compute.internal    Ready    metrics,node                           7d    v1.20.4-eks-6b7464
ip-10-225-49-197.us-west-2.compute.internal   Ready    metrics,node                           7d    v1.20.4-eks-6b7464
ip-10-225-49-212.us-west-2.compute.internal   Ready    kiam,node                              7d    v1.20.4-eks-6b7464
ip-10-225-49-24.us-west-2.compute.internal    Ready    node,nodes                             7d    v1.20.4-eks-6b7464
ip-10-225-49-249.us-west-2.compute.internal   Ready    node,test5-foo5bar5-usw2-dev-default   42m   v1.20.4-eks-6b7464. <=== p instance type
ip-10-225-49-96.us-west-2.compute.internal    Ready    node,system                            7d    v1.20.4-eks-6b7464
ip-10-225-50-116.us-west-2.compute.internal   Ready    kiam,node                              7d    v1.20.4-eks-6b7464
ip-10-225-50-188.us-west-2.compute.internal   Ready    node,system                            7d    v1.20.4-eks-6b7464
ip-10-225-50-251.us-west-2.compute.internal   Ready    metrics,node                           7d    v1.20.4-eks-6b7464
ip-10-225-50-65.us-west-2.compute.internal    Ready    node,nodes                             7d    v1.20.4-eks-6b7464
ip-10-225-51-166.us-west-2.compute.internal   Ready    metrics,node                           7d    v1.20.4-eks-6b7464
ip-10-225-52-65.us-west-2.compute.internal    Ready    node,system                            7d    v1.20.4-eks-6b7464
ip-10-225-52-91.us-west-2.compute.internal    Ready    kiam,node                              7d    v1.20.4-eks-6b7464
ip-10-225-53-22.us-west-2.compute.internal    Ready    node,nodes                             7d    v1.20.4-eks-6b7464


2. Deploy the add-on.
# This addon.yaml is based off the Nvidia containerd operator available at https://github.com/NVIDIA/gpu-operator/tree/master/deployments/gpu-operator
# It has Intuit specific customizations for container registry images, Intuit specific node-selectors for the GPU instances and such.
# Other than the it should be similar to the objects in the nvidia public repo.

$ k apply -f ./addon.yaml
addon.addonmgr.keikoproj.io/gpu-operator created
namespace/addon-gpuoperator-ns created
serviceaccount/node-feature-discovery created
serviceaccount/gpu-operator created
configmap/nfd-worker-conf created
clusterrole.rbac.authorization.k8s.io/gpu-operator-node-feature-discovery created
clusterrole.rbac.authorization.k8s.io/gpu-operator created
clusterrolebinding.rbac.authorization.k8s.io/gpu-operator-node-feature-discovery created
clusterrolebinding.rbac.authorization.k8s.io/gpu-operator created
service/gpu-operator-node-feature-discovery-master created
daemonset.apps/gpu-operator-node-feature-discovery-worker created
deployment.apps/gpu-operator-node-feature-discovery-master created
deployment.apps/gpu-operator created
customresourcedefinition.apiextensions.k8s.io/clusterpolicies.nvidia.com created
clusterpolicy.nvidia.com/cluster-policy created


3. Switch to the GPU specific ns, 'addon-gpuoperator-ns'
$ kubens addon-gpuoperator-ns
Context "service-account" modified.
Active namespace is "addon-gpuoperator-ns".

4. Get Pods. 
# Notice that the nvidia-driver-daemonset-n7lmt pods's init container is in CrashLoopBackoff and hence blocks other pods. 
$ k get pods
NAME                                                          READY   STATUS                  RESTARTS   AGE
gpu-feature-discovery-bxz8k                                   0/1     Init:0/1                0          76s
gpu-operator-64b75fc65-kkzts                                  1/1     Running                 0          4m9s
gpu-operator-node-feature-discovery-master-54b9f995d5-6b5mg   1/1     Running                 0          4m9s
gpu-operator-node-feature-discovery-worker-j4zjx              1/1     Running                 0          4m9s
nvidia-container-toolkit-daemonset-tk5w7                      0/1     Init:0/1                0          76s
nvidia-dcgm-exporter-l57st                                    0/1     Init:0/1                0          76s
nvidia-dcgm-mkmrx                                             0/1     Init:0/1                0          76s
nvidia-device-plugin-daemonset-9md9h                          0/1     Init:0/1                0          76s
nvidia-driver-daemonset-n7lmt                                 0/1     Init:CrashLoopBackOff   4          4m5s 
nvidia-operator-validator-zcgpg                               0/1     Init:0/4                0          76s


5. Get logs

# Get the logs of nvidia-driver-daemonset-n7lmt's init-container, k8s-driver-manager
$ k logs nvidia-driver-daemonset-wjdrh -c k8s-driver-manager
Getting current value of the 'nvidia.com/gpu.deploy.operator-validator' node label
Current value of 'nvidia.com/gpu.deploy.operator-validator=true'
Getting current value of the 'nvidia.com/gpu.deploy.container-toolkit' node label
Current value of 'nvidia.com/gpu.deploy.container-toolkit=true'
Getting current value of the 'nvidia.com/gpu.deploy.device-plugin' node label
Current value of 'nvidia.com/gpu.deploy.device-plugin=true'
Getting current value of the 'nvidia.com/gpu.deploy.gpu-feature-discovery' node label
Current value of 'nvidia.com/gpu.deploy.gpu-feature-discovery=true'
Getting current value of the 'nvidia.com/gpu.deploy.dcgm-exporter' node label
Current value of 'nvidia.com/gpu.deploy.dcgm-exporter=true'
Getting current value of the 'nvidia.com/gpu.deploy.dcgm' node label
Current value of 'nvidia.com/gpu.deploy.dcgm=true'
Getting current value of the 'nvidia.com/gpu.deploy.mig-manager' node label
Current value of 'nvidia.com/gpu.deploy.mig-manager='
nvidia driver module is already loaded with refcount 17
Shutting down all GPU clients on the current node by disabling their component-specific nodeSelector labels
node/ip-10-225-49-249.us-west-2.compute.internal labeled
Waiting for the operator-validator to shutdown
pod/nvidia-operator-validator-hvdxz condition met
Waiting for the container-toolkit to shutdown
pod/nvidia-container-toolkit-daemonset-lwnlc condition met
Waiting for the device-plugin to shutdown
Waiting for gpu-feature-discovery to shutdown
Waiting for dcgm-exporter to shutdown
Waiting for dcgm to shutdown
Unloading NVIDIA driver kernel modules...
nvidia_drm             53248  0
nvidia_modeset       1187840  1 nvidia_drm
nvidia              19812352  17 nvidia_modeset
drm_kms_helper        204800  1 nvidia_drm
drm                   528384  3 drm_kms_helper,nvidia_drm
i2c_core               90112  3 drm_kms_helper,nvidia,drm
Could not unload NVIDIA driver kernel modules, driver is in use
Unable to cleanup driver modules, attempting again with node drain...
Draining node ip-10-225-49-249.us-west-2.compute.internal...
node/ip-10-225-49-249.us-west-2.compute.internal cordoned
node/ip-10-225-49-249.us-west-2.compute.internal drained
WARNING: ignoring DaemonSet-managed Pods: addon-gpuoperator-ns/gpu-operator-node-feature-discovery-worker-j4zjx, addon-gpuoperator-ns/nvidia-driver-daemonset-wjdrh, addon-oil-ns/o11y-ds-bsg45, addon-wavefrontcollectors-ns/wavefront-collector-r9nkv, falco-ns/falco-daemonset-jwx96, kube-system/aws-node-kzgjf, kube-system/calico-node-24xms, kube-system/kiam-agent-n555v, kube-system/kube-proxy-x4kkp, kube-system/node-local-dns-whmvh
Unloading NVIDIA driver kernel modules...
nvidia_drm             53248  0
nvidia_modeset       1187840  1 nvidia_drm
nvidia              19812352  17 nvidia_modeset
drm_kms_helper        204800  1 nvidia_drm
drm                   528384  3 drm_kms_helper,nvidia_drm
i2c_core               90112  3 drm_kms_helper,nvidia,drm
Could not unload NVIDIA driver kernel modules, driver is in use
Uncordoning node ip-10-225-49-249.us-west-2.compute.internal...
node/ip-10-225-49-249.us-west-2.compute.internal uncordoned
Rescheduling all GPU clients on the current node by enabling their component-specific nodeSelector labels
node/ip-10-225-49-249.us-west-2.compute.internal labeled