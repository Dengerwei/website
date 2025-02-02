# CPU QoS

## Introduction

Kubernetes allows you to deploy various types of containerized applications on the same node. This causes applications with different priorities to compete for CPU resources. As a result, the performance of the applications with high priorities cannot be guaranteed. Koordinator allows you to use quality of service (QoS) classes to guarantee CPU resources for applications with high priorities. This topic describes how to configure the CPU QoS feature for pods.

## Background

To fully utilize computing resources, workloads of different priorities are usually deployed on the same node. For example, latency-sensitive (LS) workloads (with high priorities) and best-effort (BE) workloads (with low priorities) can be deployed on the same node. However, this may cause these workloads to compete for computing resources. In Kubernetes, CPU requests and CPU limits are used to control the amount of CPU resources that pods can use. However, pods may still compete for CPU resources. For example, BE pods and LS pods can share CPU cores or vCPU cores. When the loads of the BE pods increase, the performance of the LS pods is compromised. As a result, the response latency of the application that uses the LS pods increases.

To reduce the performance impact on the BE pods in this scenario, you can use the CPU QoS feature provided by Koordinator to limit the CPU usage of the LS pods. The CPU QoS feature is based on Alibaba Cloud Linux 2. Koordinator allows you to use the group identity feature available in Alibaba Cloud Linux 2 to configure Linux scheduling priorities for pods. In an environment where both LS pods and BE pods are deployed, you can set the priority of LS pods to high and the priority of BE pods to low to avoid resource contention. The LS pods are prioritized to use the limited CPU resources to ensure the service quality of the corresponding application. For more information, see [Group identity feature](https://www.alibabacloud.com/help/en/elastic-compute-service/latest/group-identity-feature).

You can gain the following benefits after you enable the CPU QoS feature:

- The wake-up latency of tasks for LS workloads is minimized.
- Waking up tasks for BE workloads does not adversely impact the performance of LS pods.
- Tasks for BE workloads cannot use the simultaneous multithreading (SMT) scheduler to share CPU cores. This further reduces the impact on the performance of LS pods.

## Setup

### Prerequisites

- Kubernetes >= 1.18
- Koordinator >= 0.4
- Operating System：
  - Alibaba Cloud Linux 2（For more information, see [Group identity feature](https://www.alibabacloud.com/help/en/elastic-compute-service/latest/group-identity-feature)）
  - Anolis OS >= 8.6
  - CentOS 7.9 (Need to install the CPU Co-location scheduler plug-in from OpenAnolis community, see [here](https://koordinator.sh/blog/anolis-CPU-Co-location))

#### Prerequisites for CentOS 7.9

- Kernel: The kernel must be the official CentOS 7.9 kernel.
- version == 3.10.0
- release >= 1160.80

#### Installing plug-in

Install the plug-in:

  ```bash
  # rpm -ivh https://koordinator.sh/website/scheduler-bvt-noise-clean-$(uname -r).rpm
  ```

If you update the kernel version, you can use the following command to install the new plug-in.
  ```
  # rpm -ivh https://koordinator.sh/website/scheduler-bvt-noise-clean-$(uname -r).rpm --oldpackage
  ```

After installation, you can see the `cpu.bvt_warp_ns` in cpu cgroup directory and the usage of it is compatible with Group Identity.

#### Removing plug-in

Removing the plug-in can use the `rpm -e` command and the `cpu.bvt_warp_ns` doesn't exist either. Please make sure that no tasks are still using `cpu.bvt_warp_ns` before uninstalling.


### Installation

Please make sure Koordinator components are correctly installed in your cluster. If not, please refer to [Installation](https://koordinator.sh/docs/installation).

## Use CPU QoS

1. Create a configmap.yaml file based on the following ConfigMap content:

   ```bash
   # Example of the slo-controller-config ConfigMap.
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: slo-controller-config
     namespace: koordinator-system
   data:
     # Enable the CPU QoS feature.
     resource-qos-config: |
       {
         "clusterStrategy": {
           "lsClass": {
             "cpuQOS": {
               "enable": true,
               "groupIdentity": 2
             }
           },
           "beClass": {
             "cpuQOS": {
               "enable": true,
               "groupIdentity": -1
             }
           }
         }
       }
   ```

   Specify `lsClass` and `beClass` to assign the LS and BE classes to different pods. `cpuQOS` includes the CPU QoS parameters. The following table describes the parameters.

| Configuration item | Parameter | Valid values | Description                                                  |
| :----------------- | :-------- | :----------- | :----------------------------------------------------------- |
| `enable`           | Boolean   | truefalse    | true: enables the CPU QoS feature for all containers in the cluster.false: disables the CPU QoS feature for all containers in the cluster. |
| `groupIdentity`    | Int       | -1~2         | Specify group identities for CPU scheduling. By default, the group identity of LS pods is 2 and the group identity of BE pods is -1. A value of 0 indicates that no group identity is assigned.A greater `group identity` value indicates a higher priority in CPU scheduling. For example, you can set `cpu.bvt_warp_ns=2` for LS pods and set `cpu.bvt_warp_ns=-1` for BE pods because the priority of LS pods is higher than that of BE pods. For more information, see [Group identity feature](https://www.alibabacloud.com/help/en/elastic-compute-service/latest/group-identity-feature#task-2129392). |
   
   **Note** If `koordinator.sh/qosClass` is not specified for a pod, Koordinator configures the pod based on the original QoS class of the pod. The component uses the BE settings in the preceding ConfigMap if the original QoS class is BE. The component uses the LS settings in the preceding ConfigMap if the original QoS class is not BE

2. Check whether a ConfigMap named `slo-controller-config` exists in the `koordinator-system` namespace.

   - If a ConfigMap named  `slo-controller-config`  exists, we commend that you run the kubectl patch command to update the ConfigMap. This avoids changing other settings in the ConfigMap.

     ```sql
     kubectl patch cm -n koordinator-system slo-controller-config --patch "$(cat configmap.yaml)"
     ```

   - If no ConfigMap named `slo-controller-config`  exists, run the kubectl patch command to create a ConfigMap named ack-slo-config:

     ```undefined
     kubectl apply -f configmap.yaml
     ```

3. Create a file named ls-pod-demo.yaml based on the following YAML content:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: ls-pod-demo
     labels:
       koordinator.sh/qosClass: 'LS' # Set the QoS class of the pod to LS. 
   spec:
     containers:
     - command:
       - httpd
       - -D
       - FOREGROUND
       image: registry.cn-zhangjiakou.aliyuncs.com/acs/apache-2-4-51-for-slo-test:v0.1
       imagePullPolicy: Always
       name: apache
       resources:
         limits:
           cpu: "4"
           memory: 10Gi
         requests:
           cpu: "4"
           memory: 10Gi
     restartPolicy: Never
     schedulerName: default-scheduler
   ```

4. Run the following command to deploy the ls-pod-demo pod in the cluster:

   ```undefined
   kubectl apply -f ls-pod-demo.yaml
   ```

5. Run the following command to check whether the CPU group identity of the LS pod in the control group (cgroup) of the node takes effect:

   ```bash
   cat /sys/fs/cgroup/cpu/kubepods.slice/kubepods-pod1c20f2ad****.slice/cpu.bvt_warp_ns
   ```

   Expected output:

   ```sql
   #The group identity of the LS pod is 2 (high priority). 
   2
   ```

6. Create a file named be-pod-demo.yaml based on the following content:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: be-pod-demo
     labels:
       koordinator.sh/qosClass: 'BE' # Set the QoS class of the pod to BE. 
   spec:
     containers:
       - args:
           - '-c'
           - '1'
           - '--vm'
           - '1'
         command:
           - stress
         image: polinux/stress
         imagePullPolicy: Always
         name: stress
     restartPolicy: Always
     schedulerName: default-scheduler
     priorityClassName: koord-batch
   ```

7. Run the following command to deploy the be-pod-demo pod in the cluster:

   ```undefined
   kubectl apply -f be-pod-demo.yaml
   ```

8. Run the following command to check whether the CPU group identity of the BE pod in the cgroup of the node takes effect:

   ```python
   cat /sys/fs/cgroup/cpu/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod4b6e96c8****.slice/cpu.bvt_warp_ns
   ```

   Expected output:

   ```sql
   #The group identity of the BE pod is -1 (low priority). 
   -1
   ```

   The output shows that the priority of the LS pod is high and the priority of the BE pod is low. CPU resources are preferably scheduled to the LS pod to ensure the service quality.
