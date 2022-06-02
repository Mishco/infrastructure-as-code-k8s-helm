# Infrastructure as code (IaC), deploy using helm, Kubernetes and azure devops

## Introduction

In this article, I would like to talk about the use of infrastructure within the code base. I will guide you through the pros and cons of infrastructure as code principles.
In general, it can be used in any application or project. The main goal of this article it is to show you how it can be done and when it should be used. I used one of the project application and I skip things unrelated to this article (such as security, certificate, user management).

If you are considering using it in your project or application, remember first that it can be difficult to set up at first (it can be weeks for beginners), but it is easy (easier than trying to manage a virtual machine or the server itself) to maintain it.

## Technology overview

Among the core principles of infrastructure is controlled, tested and automatic deployment. There are several tools to manage this process. I will focus on Kubernetes and helm.

* **Kubernetes** is an open-source system for automating deployment, scaling, and management of containerized applications, mostly docker container.
With small application it can be usefully make changes in kubernetes manually, but in large application it can be necessary to have these changes managed by an automatic workflow.
In that case can help helm.

* **Helm** can help you define, install, and upgrade the complex Kubernetes application. Helm enables Kubernetes users greater control over their cluster, just like the captain of a ship at the helm.
But what if that's still not enough, you have a complex application running on kubernetes, but you would like to offer other developers to change the infrastructure of this application according to their needs, so version control (e.g.: git) can help you.

* **Azure DevOps** Server provide version control (git), project management, automated builds, testing and release management (pipelines in shortly) and other things necessary for project management. So when I put these things together, I can manage my infrastructure as code.

* **Infrastructure as code** (or IaC) is the process of managing and provisioning computer data centers through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools. Simplicity through code.

## Architecture Overview for one application

![architecture-overview](media/img/infrastructure-as-code-helm-k8s-azure_2.png)

## Jump into helm charts  

I am now little focus on helm charts and templates for them. Helm uses a packaging format called **charts**.

---
**NOTE**
I suppose that you are familiar with terms docker, container, image, cluster, and all stuff related to docker and kubernetes. If not for docker please follow [get-started](https://docs.docker.com/get-started/) and for Kubernetes visit [kubernetes-basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

---

A chart is a collection of YAML files that describe a related set of Kubernetes resources. Charts are created as files laid out in a particular directory tree. They can be packaged into versioned archives to be deployed.
A chart is organized as a collection of files inside of a directory. The directory name is the name of the chart. E.g.:

```bash
our-image/
  Chart.yaml          # A YAML file containing information about the chart
  values.yaml         # The default configuration values for this chart
  charts/             # A directory containing any charts upon which this chart depends.
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
```

Helm Templates generate manifest files, which are YAML resource descriptions that Kubernetes can understand.

In my case I use multiple images in one helm directory so structure is little different.

```bash
helm/
    first-image/
    second-image/
    third-image/
        config/
        data/
        templates/
        .helmignore
        Chart.yaml
        Charts.yaml
        values.yaml
    fourth-image/
    fifth-image/
```

---
**NOTE**
I am using Azure Cloud Shell with these version of tools:

Helm version

```bash
$ helm version --short
v3.4.0+g7090a89
```

Kubernetes version

```bash
$ kubectl version --short
Client Version: v1.20.0
Server Version: v1.17.11
```

Docker version

```bash
$ docker version
Client:
 Version:           19.03.13+azure
 API version:       1.40
 Go version:        go1.13.15
```

---

I have everything running on top of Microsoft Azure Portal. My code is stored in Azure DevOps repositories, where I also have running pipeline for all build and deployment. Also, other resources as Kubernetes service is running in Azure Portal under project subscription, and project resource group. Helm also offer connect to these resources from Azure pipelines.

Inside of running k8s (Kubernetes) cluster I have multiple namespaces, for each application one namespace so if something is going to wrong it should not affect other applications.
Firstly I can look deeply on one of the application and their templates.

In my project every application consists of one docker image (container) which is stored and build in the another repository.
Application also contains `Chart.yaml` in format:

```yaml
apiVersion: v2
name: <first-app>
description: A Helm chart for Kubernetes
type: application
version: 0.1.2
appVersion: 1.16.0
```

`values.yaml` file:

```yaml
host: first-app.full-address.com
keyvault: dev-key-vault
```

most important part is `templates` folder with multiple files:

- `helm\first-app\templates\akvs.yaml`
- `helm\first-app\templates\config.yaml`
- `helm\first-app\templates\deployment.yaml`
- `helm\first-app\templates\ingress.yaml`
- `helm\first-app\templates\service.yaml`
- `helm\first-app\templates\pvc-packages.yaml`

Store and connect secrets from azure vault `akvs.yaml`

```yaml
apiVersion: spv.no/v1alpha1
kind: AzureKeyVaultSecret
metadata:
  name: certificat-sync
  namespace: namespace-first-app
spec:
  vault:
    name: {{ .Values.keyvault }} # value get from 'values.yaml'
    object:
      name: first-app
      type: certificate
  output:
    secret:
      name: tls-secret # kubernetes secret name
      type: kubernetes.io/tls # kubernetes secret type
```

An external configuration loaded from a file that is better manageable than editing inside a kubernetes cluster. You can edit this file (*first-app-config.gcfg*) in the git repository and it will load with the latest information when changed.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: first-app-config
  labels:
    run: first-app
data:
  "first-app-config.gcfg": |-
{{  .Files.Get "config/first-app-config.gcfg" | indent 4 }}
```

Service and PVC files are not so important for our focus, but basically the service passes the default application port to web port 80. Ingress takes the rest and secures it. And PVC (PersistentVolumeClaim) is piece of storage in the cluster.

Ingress controller, basically, pass a web port for the http port and use it for this certificate, and tls make https for you.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: first-app-ingress
  namespace: namespace-first-app
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30000"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - host: {{ .Values.host }} # value get from 'values.yaml'
      http:
        paths:
          - backend:
              serviceName: first-app-service
              servicePort: 80
            path: /(.*)
  tls:
    - hosts:
        - {{ .Values.host }} # value get from 'values.yaml'
      secretName: tls-secret
```

The most important file is `deployment.yaml`. I will cut it into small pieces to make it more consistent and smooth. The first part is similar to another yaml configuration, the namespace should be the same as for the rest of the application. And one replica is enough for now, in the future you can increase it manually or scale them automatically. For automatic scaling in kubernetes cluster visit [horizontal-pod-autoscale](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) or [autoscaling-in-kubernetes](https://kubernetes.io/blog/2016/07/autoscaling-in-kubernetes/).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: first-app
  namespace: namespace-first-app
spec:
  replicas: 1
  selector:
    matchLabels:
      run: first-app
```

Next part of `deployment.yaml` file is this checksum which handle each change in config file. If any changes is catch in `config.yaml` it will restart deployment for us. So it update my application automatically.

```yaml
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/config.yaml") . | sha256sum }}
      labels:
        run: first-app
```

This part consists of few important things. First environment variables, like License can be handle by `Values` file. Docker image itself, it is useful to use tags and port for specific application the same as I specify in service.

```yaml
    spec:
      containers:
        - env:
            - name: LICENSE
              value: {{ .Values.LICENSE }}
          image: dev.azurecr.io/first-app-application:latest
          name: first-app
          ports:
            - containerPort: 3939 # Application port
              protocol: TCP
```

Persistent Volume is mapped in this part of `deployment.yaml` file, where I say to where should image find folder, files or configuration in persistent volumes.

```yaml
          resources: {}
          volumeMounts:
            - mountPath: "/data/"
              name: data
            - name: config-volume
              mountPath: "/etc/config.gcfg"
              subPath: first-app-config.gcfg
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: azurefile-share
        - name: config-volume
          configMap:
            name: first-app-config
```

At the last part of deployment file is responsible for using resource outside running containers.

```yaml
          securityContext:
            privileged: true
            allowPrivilegeEscalation: true
      restartPolicy: Always
```


## Azure DevOps pipelines  for helm template

When I have prepared all the helm charts and templates, I will focus on the pipeline. This channel should basically create or re-create the application I want to use with all the settings and file system. 

The trigger for this pipeline should be any change in the folder where all the files, templates, and helm charts associated with the first application are. Azure Devops already has task for [helm deploy](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/helm-deploy?view=azure-devops), which can make this job for me.

```yaml
trigger:
  paths:
    include:
      - '/helm/first-app'

jobs:
  - job:
    displayName: Install first-app via helm 3
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - task: HelmDeploy@0
        displayName: Helm install
        inputs:
          azureSubscriptionEndpoint: DEV-Azure-Connection
          azureResourceGroup: DEV-ResrouceGroup
          kubernetesCluster: kubernetes-cluster-on-azure
          command: upgrade
          chartType: FilePath
          chartName: first-app
          chartPath: helm/first-app
          releaseName: first-app
          install: true
          waitForExecution: false
          arguments: "--namespace first-app"
```

Similar steps apply to all applications. Each container (application) is in a separate namespace, so it can be easily managed and maintained without disturbing others. Each application has its own docker image (container) and also its own pipeline.

## Conclusion and future steps

In this article, I showed you an example of using a container application and deploying it to the infrastructure using IaC. I mainly focused on things that can be used in any infrastructure, but in general this should also be preceded by other background work, such as ingress-controller, load-balancer for the communication. Also, installing and setting up the helm is a also important, so there are definitely many ways I can continue with this story.

I would like to continue in the next articles to show you these things, the ingress-controller, load-balancer, certification and hopefully I will also focus on Azure Resource Manager (ARM) templates.

tags: `Kubernetes, Docker, Helm, Azure, Devops, Pipelines, IaaS, IaC`

## Sources

- [principles/infrastructure-as-code](https://www.atlassian.com/continuous-delivery/principles/infrastructure-as-code)
- [helm charts](https://helm.sh/docs/topics/charts/)
- [charts best practises](https://helm.sh/docs/chart_best_practices/templates/)
- [azure devops helm deploy](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/helm-deploy?view=azure-devops)
- [arm templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/)
- [horizontal-pod-autoscale](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [autoscaling-in-kubernetes](https://kubernetes.io/blog/2016/07/autoscaling-in-kubernetes/)
