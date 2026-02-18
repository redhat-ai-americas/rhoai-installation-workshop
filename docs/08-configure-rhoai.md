# 8. Administrative Configurations for RHOAI / Data Science Pipelines

<p align="center">
<a href="/docs/07-install-rhoai-operator.md">Prev</a>
&nbsp;&nbsp;&nbsp;
<a href="/docs/09-configure-distributed-workloads.md">Next</a>
</p>

### Objectives

- Ensure that OpenShift AI workload-related dependencies are configured
- Ensure that the OpenShift AI cluster is prepared for data scientist personas to operate on it

### Rationale

- OpenShift AI is not an all-inclusive platform in and of itself that has everything you need to get moving without configuration
- Some organizations may prefer to perform additional customization of their cluster before onboarding data science users

### Takeaways

- Installing OpenShift AI is not the last step in preparing for data science users

## 8.1 Verify GPU resources are available

### Objectives

- Ensure that OpenShift AI workloads are able to consume the GPUs in your cluster

### Rationale

- The GPU Operator and NFD must be properly installed for GPU resources to be advertised to the scheduler. Verifying this before onboarding data scientists avoids scheduling failures.

### Takeaways

- GPU availability is determined by the NVIDIA GPU Operator's device plugin advertising `nvidia.com/gpu` resources on nodes
- The RHOAI dashboard will automatically detect available GPUs when workbenches or serving runtimes request them

## Steps

- [ ] Verify that GPU nodes have `nvidia.com/gpu` resources available

      oc get nodes -l nvidia.com/gpu.machine -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'

> Expected output
>
> `ip-10-x-xx-xxx.us-xxxx-x.compute.internal	8`\

- [ ] Verify GPU node labels are present

      oc get nodes -l nvidia.com/gpu.machine -o jsonpath='{range .items[*]}{.metadata.labels.nvidia\.com/gpu\.product}{"\n"}{end}'

> Expected output
>
> `NVIDIA-L40S-SHARED`\


- [ ] If taints were configured in step 5, verify they are present on GPU nodes

      oc get node -l nvidia.com/gpu.machine -ojsonpath='{range .items[0].spec.taints[*]}{.key}{"\n"}{end}'

> Expected output
>
> `nvidia.com/gpu`

> [!NOTE]
> If taints are present on GPU nodes, ensure that any workloads targeting GPUs include matching tolerations. Tolerations are configured directly in workbench or serving runtime pod specifications.

- [ ] Create a Hardware Profile for NVIDIA GPUs

> [!IMPORTANT]
> RHOAI 3.x requires [Hardware Profiles](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.2/html/working_with_accelerators/working-with-hardware-profiles_accelerators) to assign accelerator resources to workloads. Hardware profiles are custom resources that define hardware configurations including resource identifiers, limits, tolerations, and node selectors. Without a hardware profile, the RHOAI dashboard cannot assign GPU resources to workbenches or model serving deployments.

1. From the RHOAI dashboard, click **Settings** -> **Environment setup** -> **Hardware profiles**
1. Click **Create hardware profile**
1. In the **Name** field, enter `NVIDIA GPU`
1. Add the following resource by clicking **Add resource**:

   **GPU**
   - **Resource label**: `GPU`
   - **Resource identifier**: `nvidia.com/gpu`
   - **Resource type**: 'Accelerator'
   - **Default**: `1`
   - **Minimum allowed**: `1`
   - **Maximum allowed**: `1` (adjust based on your cluster's GPU count per node)
   - Click **Add**

1. Edit the following resources by clicking the **Pencil Icon** on each:

   **CPU**
   - **Default**: `4`
   - **Minimum allowed**: `2`
   - **Maximum allowed**: `8`
   - Click **Update**

   **Memory**
   - **Default**: `24Gi`
   - **Minimum allowed**: `8Gi`
   - **Maximum allowed**: `48Gi`
   - Click **Update**

> [!NOTE]
> The CPU and memory values above are recommendations for a `g6e.4xlarge` instance (16 vCPUs, 64 GB RAM). Adjust the defaults, minimums, and maximums based on your node size and workload requirements. Leave headroom for the operating system, kubelet, and other cluster services.
1. Scroll down to **node selectors and tolerations**, the Workload allocation strategy
1. Click **⊕ Add toleration**
   - **Operator**: `Exists`
   - **Effect**: `NoSchedule`
   - **Key**: `nvidia.com/gpu`
   - Click **Add**
1. Click **Create hardware profile**

> [!NOTE]
> The hardware profile should now appear under **Settings** -> **Hardware profiles** in the RHOAI dashboard, and be selectable when creating workbenches or deploying models.

## 8.2 Increasing your non-GPU compute capacity

### Objectives

- Ensure that you have enough resources to run the prerequisite infrastructure to support overall RHOAI workloads

### Rationale

- The cluster configuration up to this point is likely going to be insufficient to run additional workloads like the OpenShift AI Data Science Pipelines server and its database, and an object storage provider to support various use cases

### Takeaways

- Some of the workloads that we deploy in the following sections may require more CPU than your cluster has available on non-GPU nodes, if you followed the recommendations in the prerequisites
- Scaling your non-GPU MachineSets will enable these workloads to schedule properly

## Steps

- [ ] Verify that you have a non-GPU Worker MachineSet configured. This MachineSet may have zero desired replicas, if you followed the cluster provisioning guidance.

      oc get machineset -n openshift-machine-api

> Expected output
>
> `NAME                                        DESIRED   CURRENT   READY   AVAILABLE   AGE`\
> `cluster-xxxxx-xxxxx-gpu-worker-us-east-2a   1         1         1       1           3h52m`\
> `cluster-xxxxx-xxxxx-worker-us-east-2a       0         0                             5h24m`

- [ ] Either copy the name of the non-GPU MachineSet you want to scale, or run the following command if you have the tooling available

      machineset=$(oc get machineset -n openshift-machine-api -ojson | jq -r '.items[] | select(.metadata.name | contains("gpu") | not) | .metadata.name' | head -1)

- [ ] Scale the MachineSet with the following command, or scale it in the web console

      oc scale machineset --replicas=1 -n openshift-machine-api $machineset

> Example output
>
> `machineset.machine.openshift.io/cluster-xxxxx-xxxxx-worker-us-east-2b scaled`

## 8.3 Add a custom serving runtime

### Objectives

- Enable using the Model Serving functionality with OpenShift AI using runtimes other than those that are supported by Red Hat directly as components of RHOAI

### Rationale

- Many users of OpenShift AI will require serving functionality or optimizations beyond those we provide and support.

### Takeaways

- KServe is the primary model serving platform
- OpenShift AI has out-of-the-box serving runtimes that are fully supported by Red Hat, but the model serving frameworks are useful well beyond those supported runtimes
- Serving runtimes may contain optimizations for hardware or model frameworks that are useful to leverage, even if they're not explicitly supported by Red Hat
- GitOps-based processes can define approved serving runtimes for data scientist or MLOps users to self-service

## Steps

- From RHOAI, Settings > Model resources and operations > Serving runtimes > Click **Add serving runtime**.
- Select the API protocol `REST`
- Select both model types `Predicitive Model & Generatative AI model` (you can change this in the future)
- Copy and Paste in the content from `configs/08/rhoai-add-serving-runtime.yaml`
- Click **Create**

## 8.4 Configuring Data Science Pipelines

### Objectives

- Configure an external database for use with Data Science Pipelines
- Configure Object Storage in support of Data Science Pipelines (which we will reuse for other things that require it)

### Rationale

- Best practice for Data Science Pipelines is to use an external high-availability database. Our example here is to demonstrate, using a non-HA database, how that might be accomplished
- Data Science Pipelines uses object storage to pass information between stages

### Takeaways

- Not all RHOAI systems inherit high availability from the cluster automatically
- Object Storage should be considered a basic requirement of most RHOAI use cases
- Amazon S3, OpenShift Data Foundations' Multi-Cloud Object Gateway or Ceph Rados Gateway, and partner solutions such as MinIO or Dell's PowerScale (formerly Isilon) solutions all present S3-compatible APIs suitable for use with OpenShift AI

## Steps

- [ ] Create a new project for the database

      oc project database || oc new-project database

> Expected output
>
> `...`
> `...`
> `Now using project "database" on server "https://api.cluster-xxxxx.xxxxx.sandbox3005.opentlc.com:6443".`
> `...`

- [ ] Create the database instance

> [!NOTE]
> The pipeline server's metadata service uses a client that _cannot_ handle the default `caching_sha2_password` authentication method in MySQL 8+. You must enable the older `mysql_native_password` authentication method in the MySQL server.

> [!WARNING]
> MySQL v9+ will not work with Data Science Pipelines because the `mysql_native_password` authentication method has been fully deprecated and removed. See this [blog post](https://blogs.oracle.com/mysql/post/mysql-90-its-time-to-abandon-the-weak-authentication-method) for more details.

    MYSQL_USER=user
    MYSQL_PASSWORD=user123
    MYSQL_DATABASE=pipelines

    oc new-app mysql \
      -i mysql:8.0-el9 \
      -e MYSQL_DEFAULT_AUTHENTICATION_PLUGIN=mysql_native_password \
      -e MYSQL_DATABASE=$MYSQL_DATABASE \
      -e MYSQL_USER=$MYSQL_USER \
      -e MYSQL_PASSWORD=$MYSQL_PASSWORD

> Expected output
>
> `--> Found image cd3719d (3 weeks old) in image stream "openshift/mysql" under tag "8.0-el9" for "mysql:8.0-el9"`
>
> `    MySQL 8.0`\
> `    ---------`\
> `    MySQL is a multi-user, multi-threaded SQL database server. The container image provides a containerized packaging of the MySQL mysqld daemon and client application. The mysqld server daemon accepts connections from clients and provides access to content from MySQL databases on behalf of the clients.`
>
> `    Tags: database, mysql, mysql80, mysql-80`
>
> `--> Found image 8fcde26 (7 days old) in image stream "openshift/mysql" under tag "8.0-el8" for "mysql"`
>
> `    MySQL 8.0`\
> `    ---------`\
> `    MySQL is a multi-user, multi-threaded SQL database server. The container image provides a containerized packaging of the MySQL mysqld daemon and client application. The mysqld server daemon accepts connections from clients and provides access to content from MySQL databases on behalf of the clients.`
>
> `    Tags: database, mysql, mysql80, mysql-80`
>
> `--> Creating resources ...`\
> `    deployment.apps "mysql" created`\
> `    deployment.apps "mysql-1" created`\
> `    service "mysql" created`\
> `    service "mysql-1" created`\
> `--> Success`\
> `    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:`\
> `     'oc expose service/mysql'`\
> `     'oc expose service/mysql-1'`\
> `    Run 'oc status' to view your app.`

- [ ] Wait for the database to install

      oc wait --for=jsonpath='{.status.replicas}'=1 deploy mysql -n database

> Expected output
>
> `deployment.apps/mysql condition met`

- [ ] Create a project for MinIO and set env vars.

      oc new-project minio
      MINIO_ROOT_USER=rootuser
      MINIO_ROOT_PASSWORD=rootuser123

> Expected output
>
> `Now using project "minio" on server "https://api.cluster-xxxxx.xxxxx.sandbox3005.opentlc.com:6443".`
>
> `You can add applications to this project with the 'new-app' command. For example, try:`
>
> `    oc new-app rails-postgresql-example`
>
> `to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:`
>
> `    kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.43 -- /agnhost serve-hostname`

- [ ] Configure the MinIO Helm repository

      helm repo add minio https://charts.min.io/

> Expected output
>
> `"minio" has been added to your repositories`
>
> Or, potentially
>
> `"minio" already exists with the same configuration, skipping`

- [ ] Deploy MinIO via the Helm chart in its own namespace with a bucket for pipelines

      helm install minio \
        --namespace minio \
        --create-namespace \
        --set replicas=1 \
        --set persistence.enabled=false \
        --set mode=standalone \
        --set rootUser=$MINIO_ROOT_USER,rootPassword=$MINIO_ROOT_PASSWORD \
        --set 'buckets[0].name=pipeline-artifacts,buckets[0].policy=none,buckets[0].purge=false' \
        minio/minio

> Expected output
>
> `NAME: minio`\
> `LAST DEPLOYED: Fri Oct 11 15:23:20 2024`\
> `NAMESPACE: minio`\
> `STATUS: deployed`\
> `...`

- [ ] Create data science projects for use with these pipelines

      oc new-project pipeline-test
      oc label ns pipeline-test opendatahub.io/dashboard=true

> Expected output
>
> `Now using project "pipeline-test" on server "https://api.cluster-5mgxv.5mgxv.sandbox3005.opentlc.com:6443".`
>
> `You can add applications to this project with the 'new-app' command. For example, try:`
>
> `    oc new-app rails-postgresql-example`
>
> `to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:`
>
> `    kubectl create deployment hello-node --image=registry.k8s.io/e2e-test-images/agnhost:2.43 -- /agnhost serve-hostname`
>
> `namespace/pipeline-test labeled`

- [ ] Create required secrets for pipeline server

      oc create secret generic dbpassword --from-literal=dbpassword=$MYSQL_PASSWORD -n pipeline-test
      oc create secret generic dspa-secret --from-literal=AWS_ACCESS_KEY_ID=$MINIO_ROOT_USER --from-literal=AWS_SECRET_ACCESS_KEY=$MINIO_ROOT_PASSWORD -n pipeline-test

> Expected output
>
> `secret/dbpassword created`\
> `secret/dspa-secret created`

- [ ] Create the pipeline server

> [!NOTE]
> The sample MySQL deployment does not have SSL configured so we need to add a `customExtraParams` field to disable the TLS check. For a production MySQL deployment, you can remove this parameter to enable the TLS check.

    oc apply -f configs/08/rhoai-test-pipeline-server.yaml

> Expected output
>
> `datasciencepipelinesapplication.datasciencepipelinesapplications.opendatahub.io/dspa created`

> [!NOTE]
> The pipeline server was configured with an example pipeline using the parameter `enableSamplePipeline`.

## Validation

Navigate to RHOAI dashboard -> Develop & Train -> Pipelines -> Project `pipeline-test`

You should see the `iris-training` pipeline and be able to execute a pipeline run. Use the three dots menu on the right side of the pipeline to instantiate the run. You could also navigate through the `pipeline-test` project directly.

## Validation

![](/assets/08-validation.gif)

## Automation key (Catch up)

- [ ] From this repository's root directory, run below command

```sh
./scripts/setup.sh -s 8
```

<p align="center">
<a href="/docs/07-install-rhoai-operator.md">Prev</a>
&nbsp;&nbsp;&nbsp;
<a href="/docs/09-configure-distributed-workloads.md">Next</a>
</p>
