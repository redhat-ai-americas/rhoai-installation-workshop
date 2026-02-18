# 6. Install RHOAI Dependencies

<p align="center">
<a href="/docs/05-configure-gpu-sharing-method.md">Prev</a>
&nbsp;&nbsp;&nbsp;
<a href="/docs/07-install-rhoai-operator.md">Next</a>
</p>

### Objectives

- Installing the prerequisite operators required by Red Hat OpenShift AI

### Rationale

- RHOAI requires a set of dependency operators to be installed before the RHOAI Operator itself.

### Takeaways

- These operators must be installed **before** installing the RHOAI Operator
- Each operator requires its own Namespace, OperatorGroup, and Subscription
- Some operators use `OwnNamespace` install mode (scoped OperatorGroup), while others use `AllNamespaces` mode
- cert-manager Operator is a prerequisite for Leader Worker Set and should be installed prior to this step

> [!IMPORTANT]
> Ensure the cert-manager Operator for Red Hat OpenShift is installed before proceeding. It is required by the Leader Worker Set Operator.

## 6.1 Install the JobSet Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the JobSet Operator

### Rationale

- The JobSet Operator manages large-scale, coordinated workloads like HPC and AI training jobs. It provides the ability to deploy a JobSet controller in OpenShift.


## Steps

- [ ] Create the JobSet Operator objects

      oc create -f configs/06/jobset-operator.yaml

> Expected output
>
> `namespace/openshift-jobset-operator created`\
> `operatorgroup.operators.coreos.com/jobset-operator created`\
> `subscription.operators.coreos.com/job-set created`

## 6.2 Install the OpenShift Custom Metrics Autoscaler Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Custom Metrics Autoscaler Operator

### Rationale

- Provides Kubernetes-based Event Driven Autoscaling (KEDA) for scaling workloads based on custom metrics, enabling efficient resource utilization for AI/ML inference workloads.


## Steps

- [ ] Create the Custom Metrics Autoscaler Operator objects

      oc create -f configs/06/custom-metrics-autoscaler-operator.yaml

> Expected output
>
> `namespace/openshift-keda created`\
> `operatorgroup.operators.coreos.com/openshift-custom-metrics-autoscaler-operator created`\
> `subscription.operators.coreos.com/openshift-custom-metrics-autoscaler-operator created`

## 6.3 Install the Leader Worker Set Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Leader Worker Set Operator

### Rationale

- Leader Worker Set provides an API for deploying a group of pods as a unit of replication, aimed at multi-host inference workloads where LLMs are shared and run across multiple devices on multiple nodes.


## Steps

> [!WARNING]
> Ensure the cert-manager Operator is installed and running before creating this operator.

- [ ] Create the Leader Worker Set Operator objects

      oc create -f configs/06/leader-worker-set-operator.yaml

> Expected output
>
> `namespace/openshift-lws-operator created`\
> `operatorgroup.operators.coreos.com/openshift-lws-operator created`\
> `subscription.operators.coreos.com/leader-worker-set created`

## 6.4 Install the Red Hat Connectivity Link Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Red Hat Connectivity Link Operator (Kuadrant)

### Rationale

- Provides API gateway and connectivity capabilities. The Red Hat Connectivity Link Operator is the productized version of the Kuadrant upstream project and includes Authorino for authentication/authorization of API endpoints.

### Takeaways

- The OLM package name is `kuadrant-operator`
- Available from the `community-operators` catalog source (not `redhat-operators`)
- Uses `AllNamespaces` install mode

## Steps

- [ ] Create the Connectivity Link Operator objects

      oc create -f configs/06/connectivity-link-operator.yaml

> Expected output
>
> `namespace/redhat-connectivity-link-operator created`\
> `operatorgroup.operators.coreos.com/connectivity-link-operator created`\
> `subscription.operators.coreos.com/kuadrant-operator created`

## 6.5 Install the Red Hat build of Kueue Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Kueue Operator (Red Hat build of Kueue)

### Rationale

- Kueue is a Kubernetes-native job queueing system that manages quotas and how jobs consume them. It is essential for managing distributed AI/ML workloads and resource allocation.

## Steps

- [ ] Create the Kueue Operator objects

      oc create -f configs/06/kueue-operator.yaml

> Expected output
>
> `namespace/openshift-kueue-operator created`\
> `operatorgroup.operators.coreos.com/kueue-operator created`\
> `subscription.operators.coreos.com/kueue-operator created`

## 6.6 Install the SR-IOV Network Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the SR-IOV Network Operator

### Rationale

- SR-IOV (Single Root I/O Virtualization) enables high-performance networking by allowing a physical network adapter to appear as multiple virtual adapters. This is important for AI/ML workloads that require high-bandwidth, low-latency networking such as GPU-to-GPU communication.

## Steps

- [ ] Create the SR-IOV Operator objects

      oc create -f configs/06/sriov-operator.yaml

> Expected output
>
> `namespace/openshift-sriov-network-operator created`\
> `operatorgroup.operators.coreos.com/sriov-network-operator created`\
> `subscription.operators.coreos.com/sriov-network-operator created`

## 6.7 Install the OpenTelemetry Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Red Hat build of OpenTelemetry Operator

### Rationale

- OpenTelemetry provides distributed tracing data collection for observability of AI/ML workloads, enabling monitoring of request flows across services.

### Takeaways

- The OLM package name is `opentelemetry-product`
- Uses `AllNamespaces` install mode

## Steps

- [ ] Create the OpenTelemetry Operator objects

      oc create -f configs/06/opentelemetry-operator.yaml

> Expected output
>
> `namespace/openshift-opentelemetry-operator created`\
> `operatorgroup.operators.coreos.com/opentelemetry-operator created`\
> `subscription.operators.coreos.com/opentelemetry-product created`

## 6.8 Install the Tempo Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Tempo Operator

### Rationale

- Tempo is a high-scale distributed tracing backend. It works alongside OpenTelemetry to store and query trace data, providing deep insights into AI/ML workload performance.

## Steps

- [ ] Create the Tempo Operator objects

      oc create -f configs/06/tempo-operator.yaml

> Expected output
>
> `namespace/openshift-tempo-operator created`\
> `operatorgroup.operators.coreos.com/tempo-operator created`\
> `subscription.operators.coreos.com/tempo-product created`

## 6.9 Install the Cluster Observability Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Cluster Observability Operator

### Rationale

- Provides unified observability capabilities for the cluster, enabling monitoring, alerting, and dashboarding for AI/ML workloads and infrastructure.

## Steps

- [ ] Create the Cluster Observability Operator objects

      oc create -f configs/06/cluster-observability-operator.yaml

> Expected output
>
> `namespace/openshift-cluster-observability-operator created`\
> `operatorgroup.operators.coreos.com/cluster-observability-operator created`\
> `subscription.operators.coreos.com/cluster-observability-operator created`

## 6.10 Install the Kernel Module Management Operator

### Objectives

- Creating the Namespace, OperatorGroup, and subscribing to the Kernel Module Management (KMM) Operator

### Rationale

- KMM manages, builds, signs, and deploys out-of-tree kernel modules and device plugins on OpenShift nodes. It is [required by RHOAI](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.2/html/installing_and_uninstalling_openshift_ai_self-managed/enabling-accelerators_install) for enabling accelerators such as NVIDIA GPUs.

## Steps

- [ ] Create the KMM Operator objects

      oc create -f configs/06/kmm-operator.yaml

> Expected output
>
> `namespace/openshift-kmm created`\
> `operatorgroup.operators.coreos.com/kmm-operator created`\
> `subscription.operators.coreos.com/kernel-module-management created`

## Validation

- [ ] Verify all dependency operators are installed and available

      oc get subscriptions -A

> Expected output
>
> `NAMESPACE                                  NAME                                                                  PACKAGE                                        SOURCE                CHANNEL`\
> `cert-manager-operator                      openshift-cert-manager-operator                                       openshift-cert-manager-operator                redhat-operators      stable-v1`\
> `nvidia-gpu-operator                        gpu-operator-certified                                                gpu-operator-certified                         certified-operators`\
> `openshift-cluster-observability-operator   cluster-observability-operator                                        cluster-observability-operator                 redhat-operators      stable`\
> `openshift-jobset-operator                  job-set                                                               job-set                                        redhat-operators      tech-preview-v0.1`\
> `openshift-keda                             openshift-custom-metrics-autoscaler-operator                          openshift-custom-metrics-autoscaler-operator   redhat-operators      stable`\
> `openshift-kmm                              kernel-module-management                                              kernel-module-management                       redhat-operators      stable`\
> `openshift-kueue-operator                   kueue-operator                                                        kueue-operator                                 redhat-operators      stable-v1.2`\
> `openshift-lws-operator                     leader-worker-set                                                     leader-worker-set                              redhat-operators      stable-v1.0`\
> `openshift-nfd                              nfd                                                                   nfd                                            redhat-operators      stable`\
> `openshift-opentelemetry-operator           opentelemetry-product                                                 opentelemetry-product                          redhat-operators      stable`\
> `openshift-operators                        servicemeshoperator3                                                  servicemeshoperator3                           redhat-operators      stable`\
> `openshift-sriov-network-operator           sriov-network-operator                                                sriov-network-operator                         redhat-operators      stable`\
> `openshift-tempo-operator                   tempo-product                                                         tempo-product                                  redhat-operators      stable`\
> `redhat-connectivity-link-operator          authorino-operator-stable-community-operators-openshift-marketplace   authorino-operator                             community-operators   stable`\
> `redhat-connectivity-link-operator          dns-operator-stable-community-operators-openshift-marketplace         dns-operator                                   community-operators   stable`\
> `redhat-connectivity-link-operator          kuadrant-operator                                                     kuadrant-operator                              community-operators   stable`\
> `redhat-connectivity-link-operator          limitador-operator-stable-community-operators-openshift-marketplace   limitador-operator                             community-operators   stable`

## Automation key (Catch up)

- [ ] From this repository's root directory, run below command

```sh
./scripts/setup.sh -s 6
```

<p align="center">
<a href="/docs/05-configure-gpu-sharing-method.md">Prev</a>
&nbsp;&nbsp;&nbsp;
<a href="/docs/07-install-rhoai-operator.md">Next</a>
</p>
