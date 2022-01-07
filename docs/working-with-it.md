---
hide:
  - navigation
---
# Working with it

Now in the second laboratory, we are going to see the essentials of Crossplane:

- [Managed Resources](https://crossplane.io/docs/v1.6/concepts/managed-resources.html)
- [Compositions](https://crossplane.io/docs/v1.6/concepts/composition.html)
- [Packages](https://crossplane.io/docs/v1.6/concepts/packages.html)

## Managed resources

A Managed Resource (MR) is Crossplane’s representation of a resource in an external system - most commonly a cloud provider. They are the building blocks of Crossplane.

For example, Bucket in the AWS Provider corresponds to an actual S3 Bucket in AWS. There is a one-to-one relationship and the changes on managed resources are reflected directly on the corresponding resource in the provider. You can see all available managed resources running this command ```kubectl get crds | grep aws | sort``` and the API specification in the [documentation](https://doc.crds.dev/github.com/crossplane/provider-aws).


### Creating a S3 Bucket

Now, we are going to create a Bucket and test that is working.

- Create the MR for the bucket.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: s3.aws.crossplane.io/v1beta1
    kind: Bucket
    metadata:
        name: my-bucket
    spec:
        forProvider:
            # Bucket parameters
            acl: public-read-write
            locationConstraint: us-east-1
        providerConfigRef:
            name: default
    EOF
    ```

- Get the status of the MR in K8s:

    ```bash
    kubectl get buckets.s3.aws.crossplane.io # or kubectl describe bucket -w

    ...
    NAME        READY   SYNCED   AGE
    my-bucket   True    True     2m22s
    ```

- Check that the bucket has been created in localstack.

    ```bash
    awslocal s3api list-buckets

    ...
    {
        "Buckets": [
            {
                "Name": "my-bucket",
                "CreationDate": "2022-01-05T12:14:49.000Z"
            }
        ],
        "Owner": {
            "DisplayName": "webfile",
            "ID": "bcaf1ffd86f41161ca5fb16fd081034f"
        }
    }
    ```


### Cleanup the MR

Finally, delete the resource created.

- To cleanup, delete the bucket as any other Kubernetes object.

    ```bash
    kubectl delete bucket my-bucket
    ```

- Check that the bucket has been deleted in localstack:

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

## Compositions

Crossplane Composite Resources (XR) are opinionated Kubernetes Custom Resources that are composed of Managed Resources. We often call them XRs for short.

![Crossplane Compostion](assets/images/crossplane-xrs-and-mrs.svg)

Composite Resources are designed to let you build your own platform with your own opinionated concepts and APIs without needing to write a Kubernetes controller from scratch. Instead, you define the schema of your XR and teach Crossplane which Managed Resources it should compose (i.e. create) when someone creates the XR you defined.

### Creating a Storage composition

First step, we have to create a composition for our Object Storage block so users will use it.

- Create a ```CompositeResourceDefinition``` for our Object Storage.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
        name: xdostorages.storage.acw.alvsanand.github.io
    spec:
        group: storage.acw.alvsanand.github.io
        names:
            kind: XDObjectStorage
            plural: xdostorages
        claimNames:
            kind: ObjectStorage
            plural: ostorages
        versions:
            - name: v1alpha1
              served: true
              referenceable: true
              schema:
                  openAPIV3Schema:
                      type: object
                      properties:
                          spec:
                              type: object
                              properties:
                                  parameters:
                                      type: object
                                      properties:
                                          storageName:
                                              type: string
                                      required:
                                          - storageName
                              required:
                                  - parameters
    EOF
    ```

- You can find a new CRD as part of the rest of CRDs.

    ```bash
    kubectl get crds | grep xdostorages

    ...
    xdostorages.storage.acw.alvsanand.github.io                2022-01-05T12:42:05Z
    ```

- Create a ```Composition``` for our Object Storage that will use a ```buckets.s3.aws.crossplane.io``` MR:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
        name: xdostorages.aws.storage.acw.alvsanand.github.io
        labels:
            serviceType: storage
            provider: aws
    spec:
        compositeTypeRef:
            apiVersion: storage.acw.alvsanand.github.io/v1alpha1
            kind: XDObjectStorage
        resources:
            - name: s3bucket
              base:
                  apiVersion: s3.aws.crossplane.io/v1beta1
                  kind: Bucket
                  spec:
                      forProvider:
                          acl: public-read-write
                          locationConstraint: us-east-1
                      providerConfigRef:
                          name: default
              patches:
                  - fromFieldPath: "spec.parameters.storageName"
                    toFieldPath: "metadata.name"
                    transforms:
                        - type: string
                          string:
                              fmt: "%s-acw"
    EOF
    ```

- You can find a new composition.

    ```bash
    kubectl get composition | grep xdostorages

    ...
    xdostorages.aws.storage.acw.alvsanand.github.io   27s
    ```

### Creating a Storage claim

Second step, user will create a claim for the Storage composition.

- Create a ```CompositeResourceDefinition``` for our Object Storage:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: storage.acw.alvsanand.github.io/v1alpha1
    kind: XDObjectStorage
    metadata:
        name: some-bucket
        namespace: default
    spec:
        compositionSelector:
            matchLabels:
                provider: aws
        parameters:
            storageName: some-bucket
    EOF
    ```

- Check that the xdostorages is created.

    ```bash
    kubectl get xdostorages

    ...
    NAME          READY   COMPOSITION                                       AGE
    some-bucket   True    xdostorages.aws.storage.acw.alvsanand.github.io   89s
    ```

- Check that the bucket object is created.

    ```bash
    kubectl get bucket

    ...
    NAME              READY   SYNCED   AGE
    some-bucket-acw   True    True     113s
    ```

- Check that the bucket has been created in localstack.

    ```bash
    awslocal s3api list-buckets

    ...
    {
        "Buckets": [
            {
                "Name": "some-bucket-acw",
                "CreationDate": "2022-01-05T15:36:06.000Z"
            }
        ],
        "Owner": {
            "DisplayName": "webfile",
            "ID": "bcaf1ffd86f41161ca5fb16fd081034f"
        }
    }
    ```

### Cleanup the Composition

Last step, delete all resources created.

- Delete all the resources.

    ```bash
    kubectl delete xdostorages some-bucket

    kubectl delete composition xdostorages.aws.storage.acw.alvsanand.github.io

    kubectl delete crds xdostorages.storage.acw.alvsanand.github.io
    ```

- Check that the bucket has been created in localstack.

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

## Packages

Crossplane packages are opinionated OCI images that contain a stream of YAML that can be parsed by the Crossplane package manager. Crossplane packages come in two varieties: Providers and Configurations. They vary in the types of resources they may contain in their packages.

In the last part of the laboratory, we will cover [Configurations](https://crossplane.io/docs/v1.6/concepts/packages.html#configuration-packages).

### Creating a Configuration package

Firstly, we have to create the package with the required resources in you local machine.

- Create a temporal directory for our package.

    ```bash
    PACKAGE_DIR=$(mktemp -d) && cd $PACKAGE_DIR
    ```

- Create a ```CompositeResourceDefinition``` file.

    ```bash
    cat > definition.yaml <<EOF
    apiVersion: apiextensions.crossplane.io/v1
    kind: CompositeResourceDefinition
    metadata:
        name: xdostorages.storage.acw.alvsanand.github.io
    spec:
        group: storage.acw.alvsanand.github.io
        names:
            kind: XDObjectStorage
            plural: xdostorages
        claimNames:
            kind: ObjectStorage
            plural: ostorages
        versions:
            - name: v1alpha1
              served: true
              referenceable: true
              schema:
                  openAPIV3Schema:
                      type: object
                      properties:
                          spec:
                              type: object
                              properties:
                                  parameters:
                                      type: object
                                      properties:
                                          storageName:
                                              type: string
                                      required:
                                          - storageName
                              required:
                                  - parameters
    EOF
    ```

- Create a ```Composition``` file.

    ```bash
    cat > composition.yaml <<EOF
    apiVersion: apiextensions.crossplane.io/v1
    kind: Composition
    metadata:
        name: xdostorages.aws.storage.acw.alvsanand.github.io
        labels:
            serviceType: storage
            provider: aws
    spec:
        compositeTypeRef:
            apiVersion: storage.acw.alvsanand.github.io/v1alpha1
            kind: XDObjectStorage
        resources:
            - name: s3bucket
              base:
                  apiVersion: s3.aws.crossplane.io/v1beta1
                  kind: Bucket
                  spec:
                      forProvider:
                          acl: public-read-write
                          locationConstraint: us-east-1
                      providerConfigRef:
                          name: default
              patches:
                  - fromFieldPath: "spec.parameters.storageName"
                    toFieldPath: "metadata.name"
                    transforms:
                        - type: string
                          string:
                              fmt: "%s-acw"
    EOF
    ```

- Create a ```Configuration``` file.

    ```bash
    cat > crossplane.yaml <<EOF
    apiVersion: meta.pkg.crossplane.io/v1
    kind: Configuration
    metadata:
      name: acw-storage
      annotations:
        serviceType: storage
        provider: aws
    spec:
      crossplane:
        version: ">=v1.0.0-0"
      dependsOn:
        - provider: crossplane/provider-aws
          version: ">=v0.20.0"
    EOF
    ```

- Build the package configuration.

    ```bash
    kubectl crossplane build configuration --name acw-storage
    ```

- Create a package configuration image.

    ```bash
    IMAGE_ID=$(docker load -i acw-storage.xpkg | sed 's|Loaded image ID: ||')

    docker tag $IMAGE_ID $LOCAL_IP:5432/acw/storage:v0.1.0

    docker push $LOCAL_IP:5432/acw/storage:v0.1.0
    ```

- Create the configuration in order to have XDObjectStorage available.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: pkg.crossplane.io/v1
    kind: Configuration
    metadata:
      name: acw-storage-configuration
    spec:
      package: $LOCAL_IP:5432/acw/storage:v0.1.0
      packagePullPolicy: IfNotPresent
      revisionActivationPolicy: Automatic
      revisionHistoryLimit: 1
    EOF

    ```

- Check the Configuration.

    ```bash
    kubectl get configuration acw-storage-configuration

    kubectl get crds | grep xdostorages

    kubectl get composition | grep xdostorages
    ```

### Creating a Storage claim again

Secondly, user will create a claim for the Storage composition but this time loaded from a Configuration package.

- Create a ```CompositeResourceDefinition``` for our Object Storage.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: storage.acw.alvsanand.github.io/v1alpha1
    kind: XDObjectStorage
    metadata:
        name: some-bucket
        namespace: default
    spec:
        compositionSelector:
            matchLabels:
                provider: aws
        parameters:
            storageName: some-bucket
    EOF
    ```

- Check that the xdostorages is created.

    ```bash
    kubectl get xdostorages

    ...
    NAME          READY   COMPOSITION                                       AGE
    some-bucket   True    xdostorages.aws.storage.acw.alvsanand.github.io   89s
    ```

- Check that the bucket object is created.

    ```bash
    kubectl get bucket

    ...
    NAME              READY   SYNCED   AGE
    some-bucket-acw   True    True     113s
    ```

- Check that the bucket has been created in localstack.

    ```bash
    awslocal s3api list-buckets

    ...
    {
        "Buckets": [
            {
                "Name": "some-bucket-acw",
                "CreationDate": "2022-01-06T08:05:16.000Z"
            }
        ],
        "Owner": {
            "DisplayName": "webfile",
            "ID": "bcaf1ffd86f41161ca5fb16fd081034f"
        }
    }
    ```

### Cleanup the Composition

Finally, delete all resources created.

- Delete all the resources:

    ```bash
    kubectl delete xdostorages some-bucket

    kubectl delete composition xdostorages.aws.storage.acw.alvsanand.github.io

    kubectl delete crds xdostorages.storage.acw.alvsanand.github.io

    kubectl delete configuration acw-storage-configuration
    ```

- Check that the bucket has been created in localstack.

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
