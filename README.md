### GPU Slicing

Time-slicing allows GPUs to be oversubscribed. Under the hood, CUDA time-slicing allows workloads that land on oversubscribed GPUs to interleave with one another. Each workload has access to the GPU memory and runs in the same fault domain as all the others.

Utilizing the GPU with Karpenter not only saves cost but, more importantly, also provides us with a flexible method to schedule GPU resources to our applications within the Kubernetes cluster. Since you may own tens of applications that need the GPU in different time slots, it is so important to schedule them in a more cost-effective way in the cloud.

Slicing can be used on Nvidia GPUs in two ways: software and hardware (Multi-Instance GPU, MIG).

EKS supports both modes using the Nvidia GPU Operator and Nvidia Device Plugin. MIG functionality is provided as part of the NVIDIA GPU driver for H100, A100, and A30 cores.

## Opportunities to use GPU Slicing

Here are some example workloads that can benefit from sharing GPU resources for better utilization: 

* Low-batch inference serving, which may only process one input sample on the GPU
* High-performance computing (HPC) applications, such as simulating photon propagation, that balance computation between the CPU (to read and process inputs) and GPU (to perform computation). Some HPC applications may not achieve high throughput on the GPU portion due to bottlenecks in the CPU core performance.
* Interactive development for ML model exploration using Jupyter notebooks 
* Spark-based data analytics applications, where some tasks, or the most minor units of work, are run concurrently and benefit from better GPU utilization
* Visualization or offline rendering applications that may be bursty
* Continuous integration/continuous delivery (CICD) pipelines that want to use any available GPUs for testing

## Benefits of Multi-Instance GPU

Multi-Instance GPUs are typically used for GPU-intensive applications such as HPC workloads, hyperparameter tuning, etc. They are also used for AI model training and Inference servers where high performance and higher security between processes are required.

* MIG ensures that GPU resources are fully utilized, reducing idle times and improving overall efficiency.
* MIG static partitioning of GPUs into multiple isolated instances, each with its dedicated portion of resources, including streaming multiprocessor (SM), ensuring better and predictable streaming multiprocessor (SM) quality of service (QoS).
* Dedicated portion of memory within multiple isolated instances ensures better memory QoS.
* Static partitioning eliminates error, resulting in fault containment and system stability.
* Better data protection and isolation of malicious activities, providing better security for multi-tenant setups.

## Cost optimization

Let's create a scenario for cost optimization. For example, we have four instances g4dn.2xlarge with GPU utilization around 25%.
We can run four workloads on every GPU, so we need only one instance with four vGPUs.

    g4dn.2xlarge price per hour $0.752 (prices may vary by region)
    4 instances × $0.752 × 24 hours/day × 30 days = $2165 monthly
    1 instances × $0.752 × 24 hours/day × 30 days = $541 monthly

So, we could reduce GPU instance costs by $1624 per month. 

## Performance Implications of GPU Slicing
While GPU slicing offers advantages, it can also lead to some performance issues:

* Lower performance for heavy tasks. Programs that need the full power of a GPU might run slower when the GPU is divided into slices.
* Increased latency. Sharing the GPU over time can make tasks wait longer for their turn, adding delays.
* Limited memory. Each GPU slice has less memory, which can be a problem for tasks requiring a lot of GPU memory.
* Possible resource competition. Even with isolation, tasks using the same physical GPU might still compete for resources.
* Inconsistent performance. The performance of GPU slices can change based on how much the entire GPU is being used.

## Monitoring the utilization of GPU
You could use special instruments to monitor and find under-provisioned GPU in your k8s cluster.

[DCHM exprter](https://github.com/NVIDIA/dcgm-exporter)
[Nvidia SMI](https://docs.nvidia.com/deploy/nvidia-smi/index.html)

## Configuration of Slicing

Install NodeClass and EC2NodeClass for GPU workload:

```
kubectl apply -f karpenter-soft.yaml
```

There is a taint isolating expensive hardware from the general workload:

```
    taints:
    - key: nvidia.com/gpu
    value: "true"
    effect: NoSchedule
```

Add the config map to the same namespace as the GPU operator:

```
kubectl create -n gpu-operator -f time-slicing-config-fine.yaml
```

Configure the device plugin with the config map and set the default time-slicing configuration:

```
kubectl patch clusterpolicies.nvidia.com/cluster-policy \
    -n gpu-operator --type merge \
    -p '{"spec": {"devicePlugin": {"config": {"name": "time-slicing-config-fine"}}}}'
```

The time-slicing configuration will only be applied to new nodes.

Restart daemonsets. Confirm that the gpu-feature-discovery and nvidia-device-plugin-daemonset pods restart:

```
kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset
kubectl get events -n gpu-operator --sort-by='.lastTimestamp'
```
Create the deployment with multiple replicas:

```
kubectl apply -f time-slicing-verification.yaml
```

Verify that all five replicas are running:

```
kubectl get pods
```

##References

[GPU Sharing techniques](https://www.infracloud.io/blogs/gpu-sharing-techniques-guide-vgpu-mig-time-slicing/)

[Nvidia GPU sharing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html)

[Improving GPU utilization](https://developer.nvidia.com/blog/improving-gpu-utilization-in-kubernetes/)

[Efficient access to shared GPUs](https://kubernetes.web.cern.ch/blog/2023/03/17/efficient-access-to-shared-gpu-resources-part-2/)

[GPU Sharing on Amazon EKS](https://aws.amazon.com/blogs/containers/gpu-sharing-on-amazon-eks-with-nvidia-time-slicing-and-accelerated-ec2-instances/)

[GPU Telemetry](https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/index.html)

[MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

[GPU Advanced Troubleshooting](https://superorbital.io/blog/gpu-kubernetes-nvidia-advanced-troubleshooting/)
