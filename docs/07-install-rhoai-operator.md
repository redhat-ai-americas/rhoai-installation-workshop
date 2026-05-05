# 7. Install RHOAI operator

<p align="center">
<a href="/docs/06-install-rhoai-dependencies.md">Prev</a>
&nbsp;&nbsp;&nbsp;
<a href="/docs/08-configure-rhoai.md">Next</a>
</p>

### Objectives

- Creating the Namespace, OperatorGroup and subscribing to the Red Hat OpenShift AI Operator

### Rationale

- Needed to run AI demos

### Takeaways

- RHOAI uses the `fast-3.x` channel
- Requires OpenShift Container Platform 4.19 or later
- Review the default-dsci
- Review the created projects

Before you install RHOAI, it is important to understand how its dependencies will be managed. The dependency operators should have been installed in the [previous step](/docs/06-install-rhoai-dependencies.md). Below are the required and **use-case dependent operators**:

| Operator                                        | Description                                                         |
| ----------------------------------------------- | ------------------------------------------------------------------- |
| `JobSet Operator`                               | manages large-scale coordinated AI training workloads               |
| `Custom Metrics Autoscaler Operator`            | event-driven autoscaling for AI/ML inference workloads              |
| `cert-manager Operator`                         | required by Leader Worker Set Operator                              |
| `Leader Worker Set Operator`                    | multi-host inference for sharded LLMs across multiple nodes         |
| `Red Hat Connectivity Link Operator`            | API gateway and connectivity (Kuadrant/Authorino)                   |
| `Kueue Operator`                                | job queueing and resource quota management for AI/ML workloads      |
| `SR-IOV Network Operator`                       | high-performance networking for GPU-to-GPU communication            |
| `OpenTelemetry Operator`                        | distributed tracing data collection for observability               |
| `Tempo Operator`                                | distributed tracing backend for storing and querying trace data     |
| `Cluster Observability Operator`                | unified cluster observability, monitoring, and alerting             |
| `Red Hat Node Feature Discovery (NFD) Operator` | if additional hardware features are being utilized, like GPU        |
| `NVIDIA GPU Operator`                           | if NVIDIA GPU accelerators exist                                    |
| `Kernel Module Management (KMM) Operator`       | manages out-of-tree kernel modules; required for GPU accelerators   |

## Steps

- [ ] Create the namespace in your RHOCP cluster

      oc create -f configs/07/rhoai-operator-ns.yaml

> Expected output
>
> `namespace/redhat-ods-operator created`

- [ ] Create the OperatorGroup object

      oc create -f configs/07/rhoai-operator-group.yaml

> Expected output
>
> `operatorgroup.operators.coreos.com/rhods-operator created`

> [!NOTE]
> We are using the `fast-3.x` channel. More information about supported versions, channels, and their characteristics is available [here](https://access.redhat.com/articles/rhoai-supported-configs).

- [ ] Create the Subscription object

      oc create -f configs/07/rhoai-operator-subscription.yaml

> Expected output
>
> `subscription.operators.coreos.com/rhods-operator created`

- [ ] Verify at least these projects are created `redhat-ods-applications|redhat-ods-monitoring|redhat-ods-operator` (redhat-ods-applications-auth-provider may not appear depending on your OpenShift AI Version)

      oc get projects -w | grep -E "redhat-ods|rhods"

> Expected output
>
> `redhat-ods-applications                                                           Active`\
> `redhat-ods-applications-auth-provider                                             Active`\
> `redhat-ods-monitoring                                                             Active`\
> `redhat-ods-operator                                                               Active`

> [!NOTE]
> Exit out (CTRL+C) of the above command when you see the expected output

- When you install the RHOAI Operator in the OpenShift cluster, the following new projects are created:
  1. `redhat-ods-applications` contains the dashboard and other required components of OpenShift AI. Additionally, this is where the included notebook images are stored as `ImageStreams`.
  1. `redhat-ods-applications-auth-provider` is where Authorino would be configured to run, in support of authenticating KServe model inference endpoints.
  1. `redhat-ods-monitoring` contains services for monitoring.
  1. `redhat-ods-operator` contains the RHOAI Operator itself.
  1. `rhods-notebooks` is a namespace that will get created later, where an individual user notebook environments are deployed by default. You or your data scientists must create additional projects for the applications that will use your machine learning models.

> [!IMPORTANT]
> Do not install independent software vendor (ISV) applications in namespaces associated with OpenShift AI.

- [ ] Verify `default-dsci` yaml file

      oc describe DSCInitialization -n redhat-ods-operator

> Expected output
>
> `Name:         default-dsci`\
> `...`\
> `...`\
> `...`\
> `  Phase:  Ready`\
> `  Release:`\
> `    Name:     OpenShift AI Self-Managed`\
> `    Version:  3.2.0`
> `Events:       <none>`\

## 7.1 Install RHOAI components

### Objectives

- Defining the components we want to use

### Rationale

- In order to use the RHOAI Operator, you must create a DataScienceCluster instance.

### Takeaways

- The Channel, DSC and DSCI are critical
- The DSC uses the `datasciencecluster.opendatahub.io/v2` API version
- The DSC defines which components are `Managed`, `Removed`, or `Unmanaged`

#### RHOAI Component States

There are 3x RHOAI Operator dependency states to be set: `Managed`, `Removed`, and `Unmanaged`.

| State       | Description                                                                                                                                                                                                                                                                                                                            |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Managed`   | The RHOAI Operator manages the dependency. RHOAI manages the operands, not the operators.                                                                                                                                                                                                                                              |
| `Removed`   | The RHOAI Operator removes the dependency. Changing from `Managed` to `Removed` does remove the dependency                                                                                                                                                                                                                             |
| `Unmanaged` | The RHOAI Operator does not manage the dependency allowing for an administrator to manage it instead. Changing from `Managed` to `Unmanaged` does not remove the dependency. For example, this is important when the customer has an existing configuration. It won't create it when it doesn't exist, but you can make manual changes. |

## Steps

> [!NOTE]
> In order to use RHOAI functionality, you must create a DataScienceCluster instance.
> The provided DSC sets all components to `Removed` by default. Update the `managementState` to `Managed` for the components you want to enable.

- [ ] Create the DSC object

      oc create -f configs/07/rhoai-operator-dsc.yaml

> Expected output
>
> `datasciencecluster.datasciencecluster.opendatahub.io/default-dsc created`

- [ ] Wait for the DSC to show Ready

> [!NOTE]
> This may take up to around ten minutes.

    oc wait --for=jsonpath='{.status.phase}'=Ready datasciencecluster default-dsc --timeout=15m

> Expected output
>
> `datasciencecluster.datasciencecluster.opendatahub.io/default-dsc condition met`

- [ ] Depending on the OpenShift Cluster you may need to scale your deployment to one

      oc patch deployment rhods-dashboard -n redhat-ods-applications --type=merge -p '{"spec":{"replicas":1}}'

- [ ] Verify DSC and related object creation

      oc get DataScienceCluster,DSCInitialization -n redhat-ods-operator

> Expected output
>
> `NAME                                                               R`\
> `datasciencecluster.datasciencecluster.opendatahub.io/default-dsc   4m31s`
>
> `NAME                                                              AGE     PHASE   CREATED AT`\
> `dscinitialization.dscinitialization.opendatahub.io/default-dsci   7m43s   Ready   2026-02-11T17:49:37Z`

## Validation

![](/assets/07-validation.gif)

## Automation key (Catch up)

- [ ] From this repository's root directory, run below command

```sh
./scripts/setup.sh -s 7
```

<p align="center">
<a href="/docs/06-install-rhoai-dependencies.md">Prev</a>
&nbsp;&nbsp;&nbsp;
<a href="/docs/08-configure-rhoai.md">Next</a>
</p>
