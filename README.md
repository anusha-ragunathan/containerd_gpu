Enabling GPU workloads with containerd as CRI in a Kubernetes cluster

<b>Generating nvidia_gpu_operator.yaml.orig </b>


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

 <b>Customize the above and generate nvidia_gpu_operator.yaml</b>

 a. Add nodeSelector and tolerations to the Daemonsets, so that the pods are scheduled on the appropriate P instance nodes.

b. Disable nvidia driver install, since our AMIs pre-bake the drivers. 

