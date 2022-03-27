---
hide:
  - navigation
---
# Lab 1 - Setup

In the first lab, you will setup your computer for the following laboratories. Here are some services that we will install:

- Local Kubernetes cluster
- Crossplane
- [LocalStack](https://localstack.cloud/)

## 0. Requisites

Before begining with this laboratory, it is neccessary to have the installed and configured the following elements:

- Linux, WSL or Linux VM.
- [Docker](https://docs.docker.com/engine/install/).
- [Helm](https://helm.sh/)
- [Github Client](https://github.com/cli/cli)

## 1. Kubernetes

![k3d](assets/images/k3d.svg){ style="width:400px" align=right }

Although there are many easy of ways for deploying a Kubernetes cluster ([Docker Desktop](https://www.docker.com/products/docker-desktop), [Minikube](https://minikube.sigs.k8s.io/docs/), [Kind](https://kind.sigs.k8s.io/), [Rancher Desktop](https://rancherdesktop.io/), ...), we have selected [k3d](https://k3d.io/) because it is ver fast and easy to install.

!!! info
    [k3d](https://k3d.io/) is a lightweight wrapper to run k3s (Rancher Lab’s minimal Kubernetes distribution) in docker.

    k3d makes it very easy to create single- and multi-node k3s clusters in docker, e.g. for local development on Kubernetes.

- [Install](https://k3d.io/stable/#quick-start) k3d cli.

    ```bash
    curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
    ```

- Create a k3d cluster with 1 master and 2 workers with a registry.

    ```bash
    k3d cluster create --agents 2 --servers 1 --registry-create k3d-local-registry:0.0.0.0:5432
    ```

- Obtain the local IP address of your computer.

    ```bash
    LOCAL_IP=$(ip -o route get to 8.8.8.8 | sed -n 's/.*src \([0-9.]\+\).*/\1/p')
    ```

- [Add](https://stackoverflow.com/a/63227959) "LOCAL_IP:5432" to "insecure-registries" in Docker.

!!! warning
    This is necessary, so we can use the Image registry created by k3d.

- When you have finished the workshop, you should delete the Kubernetes cluster.

    ```bash
    k3d cluster delete
    ```

## 2. Crossplane

- Install Crossplane.

    ```bash
    kubectl create namespace crossplane-system

    helm repo add crossplane-stable https://charts.crossplane.io/stable
    helm repo update

    helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
    ```

- Wait until Crossplane is ready.

    ```bash
    kubectl get all -n crossplane-system

    ...
    NAME                                           READY   STATUS    RESTARTS   AGE
    pod/crossplane-rbac-manager-5bf768f5dc-m5fvr   1/1     Running   0          34s
    pod/crossplane-7545d9567-wntct                 1/1     Running   0          34s

    NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/crossplane-rbac-manager   1/1     1            1           34s
    deployment.apps/crossplane                1/1     1            1           34s

    NAME                                                 DESIRED   CURRENT   READY   AGE
    replicaset.apps/crossplane-rbac-manager-5bf768f5dc   1         1         1       34s
    replicaset.apps/crossplane-7545d9567                 1         1         1       34s
    ```

- Install Crossplane cli.

    ```bash
    curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
    ```    

- Install AWS provider.

    ```bash
    kubectl crossplane install provider crossplane/provider-aws:v0.21.0
    ```

- Wait until AWS provider is ready.

    ```bash
    kubectl get providers.pkg.crossplane.io

    ...
    NAME                      INSTALLED   HEALTHY   PACKAGE                           AGE
    crossplane-provider-aws   True        True      crossplane/provider-aws:v0.21.0   4m38s
    ```

- Create AWS credentials for crossplane.

    ```bash
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: localstack-creds
      namespace: crossplane-system
    type: Opaque
    data:
      credentials: W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkID0gdGVzdAphd3Nfc2VjcmV0X2FjY2Vzc19rZXkgPSB0ZXN0Cg==
    ---
    apiVersion: aws.crossplane.io/v1beta1
    kind: ProviderConfig
    metadata:
      name: default
    spec:
      endpoint:
        hostnameImmutable: true
        url:
          type: Static
          static: http://localstack.awslocal.svc.cluster.local:4566
      credentials:
        source: Secret
        secretRef:
          namespace: crossplane-system
          name: localstack-creds
          key: credentials
    EOF
    ```

- List all Crossplane CRDS available.

    ```bash
    kubectl get crds | grep aws | sort

    ...
    activities.sfn.aws.crossplane.io                           2022-01-05T12:14:06Z
    addons.eks.aws.crossplane.io                               2022-01-05T12:14:07Z
    addresses.ec2.aws.crossplane.io                            2022-01-05T12:14:10Z
    aliases.kms.aws.crossplane.io                              2022-01-05T12:14:06Z
    apimappings.apigatewayv2.aws.crossplane.io                 2022-01-05T12:14:05Z
    apis.apigatewayv2.aws.crossplane.io                        2022-01-05T12:14:08Z
    authorizers.apigatewayv2.aws.crossplane.io                 2022-01-05T12:14:08Z
    backups.dynamodb.aws.crossplane.io                         2022-01-05T12:14:09Z
    brokers.mq.aws.crossplane.io                               2022-01-05T12:14:09Z
    bucketpolicies.s3.aws.crossplane.io                        2022-01-05T12:14:06Z
    buckets.s3.aws.crossplane.io                               2022-01-05T12:14:06Z
    cacheclusters.cache.aws.crossplane.io                      2022-01-05T12:14:09Z
    cachepolicies.cloudfront.aws.crossplane.io                 2022-01-05T12:14:09Z
    cachesubnetgroups.cache.aws.crossplane.io                  2022-01-05T12:14:08Z
    certificateauthorities.acmpca.aws.crossplane.io            2022-01-05T12:14:10Z
    certificateauthoritypermissions.acmpca.aws.crossplane.io   2022-01-05T12:14:05Z
    certificates.acm.aws.crossplane.io                         2022-01-05T12:14:07Z
    classifiers.glue.aws.crossplane.io                         2022-01-05T12:14:05Z
    ...
    ```

## 3. LocalStack

![LocalStack](assets/images/localstack.svg){ style="height:120px" align=left }

In order to simulate AWS cloud, we will use [LocalStack](https://localstack.cloud/).

[LocalStack](https://localstack.cloud/) computer is a cloud service emulator that runs in a single container on your laptop or in your CI environment. With LocalStack, you can run your AWS applications or Lambdas entirely on your local machine without connecting to a remote cloud provider!

- Install LocalStack helm.

    ```bash
    kubectl create namespace awslocal

    helm repo add localstack-repo https://helm.localstack.cloud
    helm repo update

    helm upgrade --install localstack localstack-repo/localstack -n awslocal
    ```

- Wait until LocalStack is ready.

    ```bash
    kubectl get all -n awslocal

    ...
    NAME                              READY   STATUS    RESTARTS   AGE
    pod/localstack-6fb5dd88d7-chkcn   1/1     Running   0          4m2s

    NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
    service/localstack   NodePort   10.43.185.234   <none>        4566:31566/TCP,4571:31571/TCP   4m2s

    NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/localstack   1/1     1            1           4m2s

    NAME                                    DESIRED   CURRENT   READY   AGE
    replicaset.apps/localstack-6fb5dd88d7   1         1         1       4m2s
    ```

- Configure LocalStack to your be accessible by your local machine.

    ```bash
    kubectl port-forward -n awslocal service/localstack 34566:4566 > /dev/null 2>&1 &

    alias awslocal="AWS_ACCESS_KEY_ID='test' AWS_SECRET_ACCESS_KEY='test' AWS_DEFAULT_REGION='us-east-1' aws --endpoint-url=http://localhost:34566"
    ```

- Test awscli.

    ```bash
    awslocal s3api list-buckets

    ...
    {
        "Buckets": [],
        "Owner": {
            "DisplayName": "webfile",
            "ID": "bcaf1ffd86f41161ca5fb16fd081034f"
        }
    }
    ```

## 4. ArgoCD

- Install ArgoCD.

    ```bash
    kubectl create namespace argocd

    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

- Wait until ArgoCD is ready.

    ```bash
    kubectl get all -n argocd

    ...
    NAME                                      READY   STATUS    RESTARTS   AGE
    pod/argocd-redis-5b6967fdfc-cjbdt         1/1     Running   0          5m4s
    pod/argocd-repo-server-656c76778f-6vz2t   1/1     Running   0          5m4s
    pod/argocd-application-controller-0       1/1     Running   0          5m3s
    pod/argocd-dex-server-66f865ffb4-twwnk    1/1     Running   0          5m4s
    pod/argocd-server-cd68f46f8-bn2fm         1/1     Running   0          5m4s

    NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    service/argocd-dex-server       ClusterIP   10.43.85.30     <none>        5556/TCP,5557/TCP,5558/TCP   5m5s
    service/argocd-metrics          ClusterIP   10.43.250.61    <none>        8082/TCP                     5m5s
    service/argocd-redis            ClusterIP   10.43.175.128   <none>        6379/TCP                     5m5s
    service/argocd-repo-server      ClusterIP   10.43.183.205   <none>        8081/TCP,8084/TCP            5m5s
    service/argocd-server           ClusterIP   10.43.35.2      <none>        80/TCP,443/TCP               5m5s
    service/argocd-server-metrics   ClusterIP   10.43.197.134   <none>        8083/TCP                     5m4s

    NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/argocd-redis         1/1     1            1           5m4s
    deployment.apps/argocd-repo-server   1/1     1            1           5m4s
    deployment.apps/argocd-dex-server    1/1     1            1           5m4s
    deployment.apps/argocd-server        1/1     1            1           5m4s

    NAME                                            DESIRED   CURRENT   READY   AGE
    replicaset.apps/argocd-redis-5b6967fdfc         1         1         1       5m4s
    replicaset.apps/argocd-repo-server-656c76778f   1         1         1       5m4s
    replicaset.apps/argocd-dex-server-66f865ffb4    1         1         1       5m4s
    replicaset.apps/argocd-server-cd68f46f8         1         1         1       5m4s

    NAME                                             READY   AGE
    statefulset.apps/argocd-application-controller   1/1     5m4s
    ```

- Install also ArgoCD CLI in you computer.

    ```bash
    sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

    sudo chmod +x /usr/local/bin/argocd
    ```

- Change ArgoCD credentials.

    ```bash
    kubectl -n argocd patch secret argocd-secret \
        -p '{"stringData": {"admin.password": "$2a$10$mivhwttXM0U5eBrZGtAG8.VSRL1l9cZNAmaSaqotIzXRBRwID1NT.",
            "admin.passwordMtime": "'$(date +%FT%T)'"
        }}'
    ```

- Expose ArgoCD UI.

    ```bash
    kubectl port-forward svc/argocd-server -n argocd 10443:443 > /dev/null 2>&1 &
    ```

- Login with ArgoCD CLI.

    ```bash
    argocd login localhost:10443 --username admin --password admin --insecure
    ```

- Finally, in order to test it, open [ArgoCD UI](https://localhost:10443) in your browser.
