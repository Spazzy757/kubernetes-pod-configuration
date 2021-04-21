# Kubernetes Configuration

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->

**Table of Contents**

- [Kubernetes Configuration](#kubernetes-configuration)
  - [Setup](#setup)
    - [Setup Kind Cluster](#setup-kind-cluster)
    - [Deploy the manifests](#deploy-the-manifests)
    - [Open a Shell in a container](#open-a-shell-in-a-container)
      - [Deployment with configuration](#deployment-with-configuration)
      - [Deployment without configuration](#deployment-without-configuration)
  - [How your Application Can be configured](#how-your-application-can-be-configured)
    - [1. Direct Configuration Environment Variables](#1-direct-configuration-environment-variables)
    - [2. Using Configmaps to get Environment Variables](#2-using-configmaps-to-get-environment-variables)
    - [3. Using Secrets to get Environment Variables](#3-using-secrets-to-get-environment-variables)
    - [4. Using Configmaps as files in container](#4-using-configmaps-as-files-in-container)
    - [5. Using Secrets as files in container](#5-using-secrets-as-files-in-container)
    - [6. Validating in the none configured pods](#6-validating-in-the-none-configured-pods)

<!-- markdown-toc end -->

## Setup

The easiest way to set this example up is to use [kind][1] and then deploy the example to a local kind cluster

### Setup Kind Cluster

_Note_: Kind Requires docker

```bash
$ kind create cluster

Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.19.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

You should now have a mini kubernetes cluster running in docker

```bash
$ kubectl get nodes

NAME                 STATUS   ROLES    AGE   VERSION
kind-control-plane   Ready    master   31m   v1.19.1
```

### Deploy the manifests

```bash
$ kubectl apply -f manifests/

configmap/config created
deployment.apps/deployment-with-config created
deployment.apps/deployment-without-config created
secret/secrets created
```

Check that the containers are running

```bash
$ kubectl get pods

NAME                                         READY   STATUS    RESTARTS   AGE
deployment-with-config-5b49d66f54-5dmbf      1/1     Running   0          49s
deployment-without-config-66b6c48dd5-6w4zg   1/1     Running   0          49s
```

### Open a Shell in a container

#### Deployment with configuration

```bash
kubectl exec -it $(kubectl get pod -l app=deployment-with-config -o name) -- bash
```

#### Deployment without configuration

```bash
kubectl exec -it $(kubectl get pod -l app=deployment-without-config -o name) -- bash
```

## How your Application Can be configured

The first thing to notice is that both pods that are running are running the same image (this can be seen in the [deployments manifests][2])

There are 5 different types of configurations that can be done in a Kubernetes/OpenShift clusters

### 1. Direct Configuration Environment Variables

You can directly set the value for an environment variable in the manifests

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-config
  labels:
    app: deployment-with-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deployment-with-config
  template:
    metadata:
      labels:
        app: deployment-with-config
    spec:
    ...
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
            ...
          env:
            - name: NORMAL_ENVIRONMENT_VARIABLE
              value: "set"
              ...
```

This will allow you to have access to this variable in the container as an Environment Variable

[Execute Into the Container](#deployment-with-configuration) and then you will be able to check the above variable is set (run the following from within the container)

```bash
root@deployment-with-config-59ff49886c-47htg:/# echo $NORMAL_ENVIRONMENT_VARIABLE
set
```

### 2. Using Configmaps to get Environment Variables

You can pull information from a [configmap][3] and set it as an environment variable

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-config
  labels:
    app: deployment-with-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deployment-with-config
  template:
    metadata:
      labels:
        app: deployment-with-config
    spec:
    ...
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
          env:
          ...
            - name: CONFIG_ENVIRONMENT_VARIABLE
              valueFrom:
                configMapKeyRef:
                  name: config
                  key: CONFIG_ENVIRONMENT_VARIABLE
                  ...
```

This will allow you to have access to this variable in the container as an Environment Variable

[Execute Into the Container](#deployment-with-configuration) and then you will be able to check the above variable is set (run the following from within the container)

```bash
root@deployment-with-config-59ff49886c-47htg:/# echo $CONFIG_ENVIRONMENT_VARIABLE
bar
```

### 3. Using Secrets to get Environment Variables

You can pull information from a [secrets][4] and set it as an environment variable, secrets are generally base64 encoded in the manifests but not in the container

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-config
  labels:
    app: deployment-with-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deployment-with-config
  template:
    metadata:
      labels:
        app: deployment-with-config
    spec:
    ...
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
          env:
          ...
          - name: SECRET_ENVIRONMENT_VARIABLE
            valueFrom:
              secretKeyRef:
                name: secrets
                key: SECRET_ENVIRONMENT_VARIABLE
```

This will allow you to have access to this variable in the container as an Environment Variable

[Execute Into the Container](#deployment-with-configuration) and then you will be able to check the above variable is set (run the following from within the container)

```bash
root@deployment-with-config-59ff49886c-47htg:/# echo $SECRET_ENVIRONMENT_VARIABLE
foo
```

### 4. Using Configmaps as files in container

You can pull information from a [configmap][3] and set it as a file in the container

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-config
  labels:
    app: deployment-with-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deployment-with-config
  template:
    metadata:
      labels:
        app: deployment-with-config
    spec:
      volumes:
      ...
        - name: config-volume
          configMap:
            name: config
            items:
              - key: game.properties
                path: game.properties
              - key: ui.properties
                path: ui.properties
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
          volumeMounts:
          ...
            - name: config-volume
              readOnly: true
              mountPath: "/config"
              ...
```

This will create a folder within the container and it will inject the two files `/config/game.properties` and `/config/ui.properties` inside the container

[Execute Into the Container](#deployment-with-configuration) and then you will be able to check the above files are set (run the following from within the container)

```bash
root@deployment-with-config-59ff49886c-47htg:/# ls -l /config
total 0
lrwxrwxrwx 1 root root 22 Apr 21 14:22 game.properties -> ..data/game.properties
lrwxrwxrwx 1 root root 20 Apr 21 14:22 ui.properties -> ..data/ui.properties

root@deployment-with-config-59ff49886c-47htg:/# cat /config/game.properties
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
```

### 5. Using Secrets as files in container

You can pull information from a [secret][4] and set it as a file in the container, secrets are generally base64 encoded in the manifests but not in the container

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-with-config
  labels:
    app: deployment-with-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deployment-with-config
  template:
    metadata:
      labels:
        app: deployment-with-config
    spec:
      volumes:
        - name: secret-volume
          secret:
            secretName: secrets
            items:
              - key: test.json
                path: test.json
                ...
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
          volumeMounts:
            - name: secret-volume
              readOnly: true
              mountPath: "/secrets"
              ...
```

This will create a folder within the container and it will inject a single file `/secrets/test.json` inside the container

[Execute Into the Container](#deployment-with-configuration) and then you will be able to check the above files are set (run the following from within the container)

```bash
root@deployment-with-config-59ff49886c-47htg:/# ls -l /secrets/
total 0
lrwxrwxrwx 1 root root 16 Apr 21 14:22 test.json -> ..data/test.json

root@deployment-with-config-59ff49886c-47htg:/# cat /secrets/test.json
{
  "foo": "bar"
}
```

### 6. Validating in the none configured pods

You can run all of the previous 5 steps in the [deployment without configuration](#deployment-without-configuration) and verify that none of this configuration is set within that container (keeping in mind it is the exact same image being used)

[1]: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
[2]: manifests/deployments.yaml
[3]: manifests/configmap.yaml
[4]: manifests/secret.yaml
