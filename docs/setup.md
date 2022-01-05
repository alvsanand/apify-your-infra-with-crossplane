---
hide:
  - navigation
  - toc
---
# Setup Crossplane

In the first lab, you will setup your computer for the following laboratories. These are the elements to install:

- Local Kubernetes cluster
- Crossplane
- [Localstack](https://localstack.cloud/)

## Requisites

You will need the following resources:

- Windows, OSX or Linux
- [Docker](https://docs.docker.com/engine/install/).
- [Helm](https://helm.sh/)

## Kubernetes cluster

In case you need a Kubernetes cluster locally, I encourage to use [k3d](https://k3d.io/) is a lightweight wrapper to run [k3s](https://github.com/rancher/k3s) (Rancher Lab’s minimal Kubernetes distribution) in docker.

- [Install](https://k3d.io/v4.4.8/#quick-start) k3d cli.

- Create a k3d cluster with 1 master and 2 workers.

    ```bash
    k3d cluster create --agents 2 --servers 1 --registry-create k3d-local-registry
    ```

- Get your machine to know k3d-local-registry:

    - If Linux based system, install ```libnss-myhostname```:

        ```bash
        sudo apt install libnss-myhostname
        ```  
    
    - For add following entry to ```/etc/hosts``` file if using Unix based system or ```C:\windows\system32\drivers\etc\hosts``` file if Windows system: 

        ```bash
        127.0.0.1 k3d-local-registry.localhost
        ```

## Localstack

In order to simulate AWS cloud, we will use [Localstack](https://localstack.cloud/).

- Install localstack helm.

    ```bash
    kubectl create namespace localaws

    helm repo add localstack-repo https://helm.localstack.cloud
    helm repo update

    helm upgrade --install localstack localstack-repo/localstack -n localaws
    ```

- Configure localstack to your be accessive by your local machine.

    ```bash
    kubectl port-forward -n localaws service/localstack 34566:4566 > /dev/null 2>&1 &

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

## Crossplane

- Install Crossplane.

    ```bash
    kubectl create namespace crossplane-system

    helm repo add crossplane-stable https://charts.crossplane.io/stable
    helm repo update

    helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
    ```

- Wait until Crossplane is installed.

    ```bash
    kubectl get all -n crossplane-system -w
    ```

- Install Crossplane cli.

    ```bash
    curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
    ```    

- Install AWS provider.

    ```bash
    kubectl crossplane install provider crossplane/provider-aws:v0.21.0
    ```

- Wait until AWS provider is installed.

    ```bash
    kubectl get providers.pkg.crossplane.io -w

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
          static: http://localstack.localaws.svc.cluster.local:4566
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
