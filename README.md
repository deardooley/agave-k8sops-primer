# Development and Deployment with the Agave Platform
> Shallow primer on developing the [Agave Science APIs](https://github.com/agaveplatform/science-apis).

## Requirements
* [Maven](https://maven.apache.org/)
* OpenJDK 9
* Golang 1.14.6
* protoc 3.15.8
* Docker 20.10.5
* Docker compose (bundled with Docker CLI)
* (DNS aliases)

### Maven Profiles
Behavior in the build is managed by Maven [build profiles](https://maven.apache.org/guides/introduction/introduction-to-profiles.html).

| Name | Default | Description |
|------|---------|-------------|
| agave | true | Base profile with all settings |
| dev | true | Enables unit tests |
| integration-test | false | Enables integration tests |
| remote-data-tests | false | Enables the full suite of remote data client integration tests. This increases pipeline time by about 15 minutes. |
| third-party-integration-test | false | Includes integration tests against third party cloud services (s3, IronMQ, etc). This increases pipeline time by about 10 minutes and incurs cloud costs. |
| coverage | false | Enables coverage analysis  to run on every service | 
| jenkins | false | @deprecated |
| publish | false | @deprecated |

### Unit Tests
```
mvn clean test
```

> _**NOTE**: to run unit tests on a single service, you may need to run install first to ensure the package dependencies are present._

### Integration Tests

```
mvn clean verify
```

> _**NOTE**: to run unit tests on a single service, you may need to run install first to ensure the package dependencies are present._

#### Test Images
Several images are required for the full suite of integration tests. Specifically, mariadb, migrations, and the go relay services. These are built as part of the `agave-migration` module build by default. Simply running with the integration-tests profile enabled will ensure all images are present.  

#### Docker Compose
Integration tests require a full support infrastructure to be present. We dynamically create this infrastructure during Maven's `pre-integration-test` lifecycle phase of each module build using Docker Compose. To ensure test sanity, we tear down the containers during the `post-integration-test` phase. This can be resource intensive for several modules, so we built in a pause after each module to allow the Docker daemon time to cleanup. We also built in a liveness check after starting the containers to ensure they come up and are functionally responsive.

When running the tests natively from your development system, the test containers will not be addressable by default. Adding the following entry to your system `/etc/hosts` file will provide consistent access to the test containers whether running inside or outside of Docker. 

```
127.0.0.1 test-proxy
```

Alternatively, you can set the following Maven properties to `localhost` if you will only be running Maven commands outside of Docker: `sftp.host`, `sftptunnel.host`, `irods.host`, `irods.host`, `irods.host`, `condor.host`, `ssh.host`, `loadleveler.host`, `lsf.host`, `moab.host`, `pbs.host`, `gridengine.host`, `slurm.host`, `torque.host`.

## Deployment
There are several ways to deploy the Core Services. In each scenario, the deployment only represents the "backend" of the Platform. This, however, is sufficent for development and testing the core services. Going forward, the roadmap is to deploy and develop exclusively for a Kubernetes environment. 

### Jenkins

CI/CD is set up using Multibranch Jenkins Plugin. This auto-discovers new branches in a repo and creates Jenkins Pipeline jobs for every repo with a Jenkinsfile at the prescribed location. Every time you push to Jenkins, the full suite of unit and integration tests will run. You don't need to do anything for this to work in your branch. However, only the protected branches in the repo (master, qa, and production) are set up to automatically deploy to k8s.  

### Docker Compose (Locally)

The core services can be run locally using the bundled Docker Compose files found in the `compose/docker-compose.core.yml`, `docker-compose.migrations.yml`, `docker-compose.persistence.yml` files. While this may be convenient at first, the large resource requirements to run it make it prohibitively difficult to maintain as a local dev environment. We will be deprecating this stack in the near future in favor of [minikube](https://minikube.sigs.k8s.io/docs/), [k3s](https://rancher.com/docs/k3s/latest/en/), [microk8s](https://microk8s.io/), or the like. 

### Helm (K8S only)

[Helm](https://helm.sh) is the defacto package manager for Kubernetes. It provides a lightweight templating engine base on Go Templating, an opinionated packaging structure, and the basic metadata needed to manage dependencies and releases. The final output of an application packaged with Helm is a compressed tarball, referred to as a "Helm Chart." Helm charts can be deployed from their unpackaged, raw directory layout, a chart on the local file system, or by referencing the chart by URL. 

Helm has a few conventions around how it discovers helm charts online. Similar to the way Maven repositories impose a directory structure, so too does Helm. While you will find quite a few people simply using GitHub to host their Helm Charts, there are also a few popular Helm Repository servers that handle the publishing, validation, archiving, and access controll around their distribution. We host our project charts and their dependencies at [https://helm.agaveplatform.org]() using the popular [Chart Museum](https://chartmuseum.com/). 

We distribute the entire Agave Platform primarily with 2 charts: [core-services Helm chart](https://github.com/agaveplatform/helm-charts) and [auth-services Helm chart](https://github.com/agaveplatform/helm-charts). Today, we will talk about the [core-services Helm chart](https://github.com/agaveplatform/helm-charts).  

#### Chart Contents
> All you want and need to know about Chart structure is available in the [Chart guide](https://docs.helm.sh/docs/topics/charts/)

Before we do anything, let's add the repo to our local helm cache
```bash
$ helm repo add agave https://helm.agaveplatform.org
```

Now we can fetch charts from the repo

```bash
$ helm fetch agave/core-services --destination ./k8sops-training/helm --untar
```

Every helm chart has the following structure
```yaml
core-services/
  .helmignore # standard ignore file with files and directories to ignore during packaging and deployment
  Chart.yaml # chart metadata
  values.yaml # global and shared variable definitions
  charts/  # contain individual core services
  templates/ # helm template files for the current chart
```

Looking at our chart templates directory, we define some secrets and configs for our db and queue. We create a PVC to share across services as a `scratch` space, and we create a configmap with the settings common to all core services, and a single Certificate CRD for [cert-manager](https://cert-manager.io/) to generate our SSL cert. 

The `Chart.yaml` describes our service metadata and includes a list of other Helm charts upon which this one relies. We ship most of those charts as subcharts bundled together with the parent, which is perfectly fine. Other charts are listed as dependencies, but are not present in our `charts` directory. Like all dependency managers, Helm will resolve and fetch these charts as part of a normal chart install or upgrade. We can force this behavior up front in case we need to reference the contents while we're developing our chart.

```bash
$ helm dependency update k8sops-training/helm/core-services/
```
You should see output resembling

```
Output

Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "gethue" chart repository
...Successfully got an update from the "elastic" chart repository
...Successfully got an update from the "appscode" chart repository
...Successfully got an update from the "gpu-helm-charts" chart repository
...Successfully got an update from the "nats" chart repository
...Successfully got an update from the "ceph-csi" chart repository
...Successfully got an update from the "geek-cookbook" chart repository
...Successfully got an update from the "agave" chart repository
...Successfully got an update from the "datamachines" chart repository
...Successfully got an update from the "grafana" chart repository
...Successfully got an update from the "gitlab" chart repository
...Successfully got an update from the "loki" chart repository
...Successfully got an update from the "prometheus-community" chart repository
...Successfully got an update from the "bitnami" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 12 charts
Downloading mariadb from repo https://charts.bitnami.com/bitnami
Dependency apps did not declare a repository. Assuming it exists in the charts directory
Dependency files did not declare a repository. Assuming it exists in the charts directory
Dependency jobs did not declare a repository. Assuming it exists in the charts directory
Dependency metadata did not declare a repository. Assuming it exists in the charts directory
Dependency monitors did not declare a repository. Assuming it exists in the charts directory
Dependency notifications did not declare a repository. Assuming it exists in the charts directory
Dependency postits did not declare a repository. Assuming it exists in the charts directory
Dependency systems did not declare a repository. Assuming it exists in the charts directory
Dependency tags did not declare a repository. Assuming it exists in the charts directory
Dependency tenants did not declare a repository. Assuming it exists in the charts directory
Dependency uuids did not declare a repository. Assuming it exists in the charts directory
Deleting outdated charts
```

#### Install

We will work out of your own namespace for this tutorial. Go ahead and set up your environment now:

$NAMESPACE
```bash
export NAMESPACE=<your username>
export AGAVE_VERSION=15bfa22
```

Now you can deploy to chart to your namespace.

```
helm install core-services "$DIR/core-services" \
    --namespace $NAMESPACE \
    --create-namespace \
    --dependency-update \
    --set global.tenantBaseUrl="${NAMESPACE}.dev.k8s.agaveplatform.org" \ 
    --set global.baseUrl="core-${NAMESPACE}.dev.k8s.agaveplatform.org" \
    --set global.agaveVersion=${AGAVE_VERSION}
```

#### Upgrade

```
helm upgrade --install core-services "$DIR/core-services" \
    --namespace $NAMESPACE \
    --set global.tenantBaseUrl="${NAMESPACE}.dev.k8s.agaveplatform.org" \ 
    --set global.baseUrl="core-${NAMESPACE}.dev.k8s.agaveplatform.org" \
    --set global.agaveVersion=${AGAVE_VERSION} \
    --set metadata.replicaCount=2
```

#### Test

```
helm test core-services --namespace $NAMESPACE  --logs
```

#### Remove

```
helm delete core-services --namespace $NAMESPACE
```

## Telepresence 

[Telepresence](https://www.getambassador.io/docs/telepresence/latest/quick-start/) is a tool that enables you to code and test microservices locally against a remote Kubernetes cluster. Like `kubectl port-forward`, but bidirectional traffic + volume migration + environment + multiple ports and services... 

> _**NOTE**: osxfuse (now macfuse) is required to handle volume forwarding from K8S to your local system. You can install this via homebrew `brew install osxfuse`.

### Connecting

```bash
telepresence connect
```

You’ll see the following output:  

```bash
Output

Connecting to traffic manager...
Connected to context agaveprod (https://149.165.157.27:6443)
```
Verify that Telepresence is working properly by connecting to the Kubernetes API server with the `status` command:

```bash
telepresence status
```
You will see the following output. `Telepresence Proxy: ON` indicates that Telepresence has configured a proxy to access services on the cluster.  

```bash
Output

Root Daemon: Running
  Version     : v2.2.0 (api 3)
  Primary DNS : ""
  Fallback DNS: ""
User Daemon: Running
  Version           : v2.2.0 (api 3)
  Ambassador Cloud  : Logged out
  Status            : Connected
  Kubernetes server : https://149.165.157.27:6443
  Kubernetes context: agaveprod
  Telepresence proxy: ON (networking to the cluster is enabled)
  Intercepts        : 0 total
```
Now you can access services running in K8S using their qualified Service names. For example, to make a request to the apps api running in your namespace, you can do the following:

```bash
curl http://apps.$NAMESPACE/
```

Similarly, say you want to query mysql, you could open your ide and connect to `core-services-mysql.2-2-27` using the username and password provided to your helm chart. You could also launch a db container with host networking enabled and query directly from there. 

```bash
docker run -it --rm --network host agave-mariadb:2.2.27 \
    mysql agavecore \
      -hcore-services-mysql.$NAMESPACE -uagaveuser -ppassword -e "show tables";
```

As a developer, this means you have to leverage databases, queues, etc, running in the cloud from your local system instead of spinning up and tearing down container stacks all the time. You may find this saves you some time. Just be aware that when you do this, the data won't be wiped between runs. Your K8S stack will persist until you manually change it. 

### Container forwarding

You can also route traffic from a specific k8s service directly to a container on your development machine. Below, we have Telepresence create a tunnel for the jobs api running in your namespace to a container it will start up for you locally. Unlike before, only the container will inherit the networking.

```bash
telepresence intercept core-services-jobs \
    --namespace $NAMESPACE \
    --port 80 \
    --env-file agavedev-telepresence-jobs.env \
    --docker-run -i -t \
        --env-file agavedev-telepresence-jobs.env \
        -e ENABLE_REMOTE_DEBUG=1 \
        -p 80:80 \
        -p 10001:10001 \
        -p 10002:10002 \
        -p 52911:52911 \
        agaveplatform/jobs-api:$AGAVE_VERSION
```

### Port forwarding

***TODO: More of the same. It's an exercise.***


### Handling volumes

***TODO: This is important, but we'll leave it for later.***

## Auth Services
To deploy a full version of the Agave Platform, you will need to install the `agave/auth-services` Helm chart in the same namespace as you deployed the `agave/core-services` Chart. 

### Configuration

A valid sandbox tenant will deploy out of the box using the default settings. For external accessibility, you should edit the default Helm chart values with base url, tenant, and email settings to match your environment.

#### Tenant

The externally accessible domain name must be provided to the tenant for the chart to deploy and work properly. The domain name will be used to create a Kubernetes Ingress resource and ssl cert for your tenant. Updated the following fields for your environment and save to a file named `auth-services-values.yml`.  

> _**NOTE:** The Helm chart assumes you have [cert-manager](https://cert-manager.io/) installed in your cluster. To disable this behavior, set `ingress.certManager.enable: "false"` and provide a set of ssl certificates in a Secret named `{{ .Release.Name }}-cert-tls` in the target $NAMESPACE._

```yaml
global:
  baseUrl: agave.example.com
  
tenant:
  name: Agave Tenant
  baseUrl: agave.example.com
  contact:
    - name: Agave Admin
      email: admin@example.com
```

#### Email
An internal [maildev]() relay server is used by default. In order for email to reach external users, you will have to configure an external SMTP server such as Gmail, Outlook, SendGrid, MailGun, Amazon SES, etc. Update the Helm chart's `email.smtp` block with the connection information for your SMTP server and append to your `auth-services-values.yml` file.

> _**NOTE:** For many commercial email providers, you will have to verify your domain either via DNS or offline mechanisms to avoid aggress junk mail filtering, and source filters from your mail provider._

```yaml
email:
  smtp:
    host: <your smtp server hostname/ip address>
    auth: "true"
    port: 587
    username: <your username/email handle>
    password: <your password/api key>
    fromName: Agave Notifier
    fromAddress: "no-reply@example.com"
```

### Helm Install

Once again, we will work out of your own namespace for this tutorial. Unlike with the core-services chart, we will pass our custom values the `helm install` command as a yaml file. We will install the `agave/auth-services` chart in the same `$NAMESPACE` as the `agave/core-services` chart.

```
helm install auth-services "$DIR/auth-services" \
    --namespace $NAMESPACE \
    --dependency-update \
    --values auth-services-values.yml
```
