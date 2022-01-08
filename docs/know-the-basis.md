---
hide:
  - navigation
---
# Lab 2 - Know the basis

In the second laboratory, we are going to see the essentials of Crossplane:

- [Managed Resources](https://crossplane.io/docs/v1.6/concepts/managed-resources.html)
- [Compositions](https://crossplane.io/docs/v1.6/concepts/composition.html)
- [Packages](https://crossplane.io/docs/v1.6/concepts/packages.html)

## 1. Managed resources

![Managed Resources](assets/images/crossplane-mrs.png){ style="width:500px" align=right }

A Managed Resource (MR) is Crossplane’s representation of a resource in an external system - most commonly a cloud provider. They are the building blocks of Crossplane.

For example, Bucket in the AWS Provider corresponds to an actual S3 Bucket in AWS. There is a one-to-one relationship and the changes on managed resources are reflected directly on the corresponding resource in the provider. You can see all available managed resources running this command ```kubectl get crds | grep aws | sort``` and the API specification in the [documentation](https://doc.crds.dev/github.com/crossplane/provider-aws).

### 1.1 Simple MR

Now, we are going to create a S3 Bucket and test that is working.

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

- Check that the bucket has been created in LocalStack.

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

### 1.3 Complex MRs

For making things a little more interesting, it is time to create several MRs that have dependencies between them. This time we are going to create several MRs that are related: ```VPC -> Subnet -> Security Group -> EC2 Instance```.

!!! tip
    In order to solve dependencies among resources created by Crossplane, you should search into the API of the providers for the attributes that ends with ```Ref``` or ```Selector```.

- Create the VPC.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: ec2.aws.crossplane.io/v1beta1
    kind: VPC
    metadata:
      name: acw-vpc
    spec:
      forProvider:
        region: us-east-1
        cidrBlock: 10.0.0.0/16
        enableDnsSupport: true
        enableDnsHostNames: true
        instanceTenancy: default
      providerConfigRef:
        name: default
    EOF
    ```

- Create a Subnet.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: ec2.aws.crossplane.io/v1beta1
    kind: Subnet
    metadata:
      name: acw-vpc-subnet1
    spec:
      forProvider:
        region: us-east-1
        availabilityZone: us-east-1b
        cidrBlock: 10.0.1.0/24
        vpcIdRef:
          name: acw-vpc
        mapPublicIPOnLaunch: true
      providerConfigRef:
        name: default
    EOF
    ```

- Create a Security Group.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: ec2.aws.crossplane.io/v1beta1
    kind: SecurityGroup
    metadata:
      name: acw-instance-sg
    spec:
      forProvider:
        region: us-east-1
        vpcIdRef:
          name: acw-vpc  
        groupName: acw-instance-sg
        description: ACW Security Group for an Instance
        egress:
          - fromPort: 443
            toPort: 443
            ipProtocol: tcp
            ipRanges:
              - cidrIp: 10.0.0.0/8
      providerConfigRef:
        name: default
    EOF
    ```

- Create a Ec2 Instance.

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: ec2.aws.crossplane.io/v1alpha1
    kind: Instance
    metadata:
      name: acw-instance
    spec:
      forProvider:
        region: us-east-1
        imageId: ami-0dc2d3e4c0f9ebd18
        securityGroupRefs:
          - name: acw-instance-sg
        subnetIdRef:
          name: acw-vpc-subnet1
      providerConfigRef:
        name: default
    EOF
    ```

- Get the status of the MRs in K8s:

    ```bash
    kubectl get vpc
    
    kubectl get subnet

    kubectl get securitygroup

    kubectl get instance
    ```

- Check that the resources has been created in LocalStack.

    ```bash
    awslocal ec2 describe-vpcs

    awslocal ec2 describe-subnets

    awslocal ec2 describe-security-groups

    awslocal ec2 describe-instances

    ...
    {
        "Reservations": [
            {
                "Groups": [],
                "Instances": [
                    {
                        "AmiLaunchIndex": 0,
                        "ImageId": "ami-0dc2d3e4c0f9ebd18",
                        "InstanceId": "i-f731721dac1e3f8fa",
                        "InstanceType": "m1.small",
                        "KernelId": "None",
                        "KeyName": "None",
                        "LaunchTime": "2022-01-08T16:33:01.000Z",
                        "Monitoring": {
                            "State": "disabled"
                        },
                        "Placement": {
                            "AvailabilityZone": "us-east-1b",
                            "GroupName": "",
                            "Tenancy": "default"
                        },
                        "PrivateDnsName": "ip-10-0-1-4.ec2.internal",
                        "PrivateIpAddress": "10.0.1.4",
                        "ProductCodes": [],
                        "PublicDnsName": "ec2-54-214-103-229.compute-1.amazonaws.com",
                        "PublicIpAddress": "54.214.103.229",
                        "State": {
                            "Code": 16,
                            "Name": "running"
                        },
                        "StateTransitionReason": "",
                        "SubnetId": "subnet-746ff808",
                        "VpcId": "vpc-55faade5",
                        "Architecture": "x86_64",
                        "BlockDeviceMappings": [
                            {
                                "DeviceName": "/dev/sda1",
                                "Ebs": {
                                    "AttachTime": "2022-01-08T16:33:01.000Z",
                                    "DeleteOnTermination": true,
                                    "Status": "in-use",
                                    "VolumeId": "vol-9f7d8500"
                                }
                            }
                        ],
                        "ClientToken": "ABCDE0000000000003",
                        "EbsOptimized": false,
                        "Hypervisor": "xen",
                        "NetworkInterfaces": [
                            {
                                "Association": {
                                    "IpOwnerId": "000000000000",
                                    "PublicIp": "54.214.103.229"
                                },
                                "Attachment": {
                                    "AttachTime": "2015-01-01T00:00:00Z",
                                    "AttachmentId": "eni-attach-23ddc790",
                                    "DeleteOnTermination": true,
                                    "DeviceIndex": 0,
                                    "Status": "attached"
                                },
                                "Description": "Primary network interface",
                                "Groups": [
                                    {
                                        "GroupName": "acw-instance-sg",
                                        "GroupId": "sg-ffeeea4d068c6649a"
                                    }
                                ],
                                "MacAddress": "1b:2b:3c:4d:5e:6f",
                                "NetworkInterfaceId": "eni-3c4ecfd4",
                                "OwnerId": "000000000000",
                                "PrivateIpAddress": "10.0.1.4",
                                "PrivateIpAddresses": [
                                    {
                                        "Association": {
                                            "IpOwnerId": "000000000000",
                                            "PublicIp": "54.214.103.229"
                                        },
                                        "Primary": true,
                                        "PrivateIpAddress": "10.0.1.4"
                                    }
                                ],
                                "SourceDestCheck": true,
                                "Status": "in-use",
                                "SubnetId": "subnet-746ff808",
                                "VpcId": "vpc-55faade5"
                            }
                        ],
                        "RootDeviceName": "/dev/sda1",
                        "RootDeviceType": "ebs",
                        "SecurityGroups": [
                            {
                                "GroupName": "acw-instance-sg",
                                "GroupId": "sg-ffeeea4d068c6649a"
                            }
                        ],
                        "SourceDestCheck": true,
                        "StateReason": {
                            "Code": "",
                            "Message": ""
                        },
                        "Tags": [
                            {
                                "Key": "crossplane-kind",
                                "Value": "instance.ec2.aws.crossplane.io"
                            },
                            {
                                "Key": "crossplane-name",
                                "Value": "acw-instance"
                            },
                            {
                                "Key": "crossplane-providerconfig",
                                "Value": "default"
                            }
                        ],
                        "VirtualizationType": "paravirtual"
                    }
                ],
                "OwnerId": "000000000000",
                "ReservationId": "r-c137140b"
            }
        ]
    }
    ```

### 1.2 Cleanup the MRs

Finally, we are going to delete the resources created before.

- To cleanup, delete all the resources using the API of Kubernetes.

    ```bash
    kubectl delete bucket my-bucket

    kubectl delete instance acw-instance
    
    kubectl delete securitygroup acw-instance-sg

    kubectl delete subnet acw-vpc-subnet1

    kubectl delete vpc acw-vpc
    ```

- Check that the resources has been deleted in LocalStack except the instances that are in state ```Terminated```:

    ```bash
    awslocal s3api list-buckets

    awslocal ec2 describe-vpcs

    awslocal ec2 describe-subnets

    awslocal ec2 describe-security-groups

    awslocal ec2 describe-instances

    ...
    {
        "Reservations": [
            {
                "Groups": [],
                "Instances": [
                    {
                        "AmiLaunchIndex": 0,
                        "ImageId": "ami-0dc2d3e4c0f9ebd18",
                        "InstanceId": "i-f731721dac1e3f8fa",
                        "InstanceType": "m1.small",
                        "KernelId": "None",
                        "KeyName": "None",
                        "LaunchTime": "2022-01-08T16:33:01.000Z",
                        "Monitoring": {
                            "State": "disabled"
                        },
                        "Placement": {
                            "AvailabilityZone": "us-east-1b",
                            "GroupName": "",
                            "Tenancy": "default"
                        },
                        "PrivateDnsName": "ip-10-0-1-4.ec2.internal",
                        "PrivateIpAddress": "10.0.1.4",
                        "ProductCodes": [],
                        "PublicDnsName": "None",
                        "State": {
                            "Code": 48,
                            "Name": "terminated"
                        },
                        "StateTransitionReason": "User initiated (2022-01-08 16:37:38 UTC)",
                        "SubnetId": "subnet-746ff808",
                        "VpcId": "vpc-55faade5",
                        "Architecture": "x86_64",
                        "BlockDeviceMappings": [],
                        "ClientToken": "ABCDE0000000000003",
                        "EbsOptimized": false,
                        "Hypervisor": "xen",
                        "NetworkInterfaces": [
                            {
                                "Attachment": {
                                    "AttachTime": "2015-01-01T00:00:00Z",
                                    "AttachmentId": "eni-attach-23ddc790",
                                    "DeleteOnTermination": true,
                                    "DeviceIndex": 0,
                                    "Status": "attached"
                                },
                                "Description": "Primary network interface",
                                "Groups": [
                                    {
                                        "GroupName": "acw-instance-sg",
                                        "GroupId": "sg-ffeeea4d068c6649a"
                                    }
                                ],
                                "MacAddress": "1b:2b:3c:4d:5e:6f",
                                "NetworkInterfaceId": "eni-3c4ecfd4",
                                "OwnerId": "000000000000",
                                "PrivateIpAddress": "10.0.1.4",
                                "PrivateIpAddresses": [
                                    {
                                        "Primary": true,
                                        "PrivateIpAddress": "10.0.1.4"
                                    }
                                ],
                                "SourceDestCheck": true,
                                "Status": "in-use",
                                "SubnetId": "subnet-746ff808",
                                "VpcId": "vpc-55faade5"
                            }
                        ],
                        "RootDeviceName": "/dev/sda1",
                        "RootDeviceType": "ebs",
                        "SecurityGroups": [
                            {
                                "GroupName": "acw-instance-sg",
                                "GroupId": "sg-ffeeea4d068c6649a"
                            }
                        ],
                        "SourceDestCheck": true,
                        "StateReason": {
                            "Code": "Client.UserInitiatedShutdown",
                            "Message": "Client.UserInitiatedShutdown: User initiated shutdown"
                        },
                        "Tags": [
                            {
                                "Key": "crossplane-kind",
                                "Value": "instance.ec2.aws.crossplane.io"
                            },
                            {
                                "Key": "crossplane-name",
                                "Value": "acw-instance"
                            },
                            {
                                "Key": "crossplane-providerconfig",
                                "Value": "default"
                            }
                        ],
                        "VirtualizationType": "paravirtual"
                    }
                ],
                "OwnerId": "000000000000",
                "ReservationId": "r-c137140b"
            }
        ]
    }
    ```

## 2. Compositions

Crossplane Composite Resources (XR) are opinionated Kubernetes Custom Resources that are composed of Managed Resources. We often call them XRs for short.

![Crossplane Compostion](assets/images/crossplane-xrs-and-mrs.svg)

Composite Resources are designed to let you build your own platform with your own opinionated concepts and APIs without needing to write a Kubernetes controller from scratch. Instead, you define the schema of your XR and teach Crossplane which Managed Resources it should compose (i.e. create) when someone creates the XR you defined.

In our laboratory, we are going to provide to the users a Object storage resource abstracting them the actual implementation that is done our current cloud.

### 2.1 XD and Composition

First step, we have to create a composition for the Object Storage.

- Create a ```CompositeResourceDefinition```.

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

- Create a ```Composition``` for that definition. It will use a ```buckets.s3.aws.crossplane.io``` MR of the AWS provider:

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

- Finally, you can find a new composition.

    ```bash
    kubectl get composition | grep xdostorages

    ...
    xdostorages.aws.storage.acw.alvsanand.github.io   27s
    ```

### 2.2 Resource Claim

Second step, user will create a claim for the Object Storage composition.

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

- Check that the S3 bucket object is created.

    ```bash
    kubectl get bucket

    ...
    NAME              READY   SYNCED   AGE
    some-bucket-acw   True    True     113s
    ```

- Check that the S3 bucket has been created in LocalStack.

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

### 2.3 Cleanup the Composition

Last step, delete all resources created.

- Delete all the resources.

    ```bash
    kubectl delete xdostorages some-bucket

    kubectl delete composition xdostorages.aws.storage.acw.alvsanand.github.io

    kubectl delete crds xdostorages.storage.acw.alvsanand.github.io
    ```

- Check that the bucket has been created in LocalStack.

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

## 3. Packages

Crossplane packages are opinionated OCI images that contain a stream of YAML that can be parsed by the Crossplane package manager. Crossplane packages come in two varieties: Providers and Configurations. They vary in the types of resources they may contain in their packages.

In the last part of the laboratory, we will cover [Configurations](https://crossplane.io/docs/v1.6/concepts/packages.html#configuration-packages).

### 3.1 Configuration Package

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

### 3.2 Resource Claim again

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

- Check that the bucket has been created in LocalStack.

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

### 3.3 Cleanup the Package

Finally, delete all resources created.

- Delete all the resources:

    ```bash
    kubectl delete xdostorages some-bucket

    kubectl delete composition xdostorages.aws.storage.acw.alvsanand.github.io

    kubectl delete crds xdostorages.storage.acw.alvsanand.github.io

    kubectl delete configuration acw-storage-configuration
    ```

- Check that the bucket has been created in LocalStack.

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
