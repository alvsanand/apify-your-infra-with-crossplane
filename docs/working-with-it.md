---
hide:
  - navigation
  - toc
---
# Working with it

Now in the second laboratory, we are going to see the essentials features of Crossplane:

- [Managed Resources](https://crossplane.io/docs/v1.6/concepts/managed-resources.html)
- [Compositions](https://crossplane.io/docs/v1.6/concepts/composition.html)
- [Packages](https://crossplane.io/docs/v1.6/concepts/packages.html)

## Managed resources

A Managed Resource (MR) is Crossplane’s representation of a resource in an external system - most commonly a cloud provider. They are the building blocks of Crossplane.

For example, Bucket in the AWS Provider corresponds to an actual S3 Bucket in AWS. There is a one-to-one relationship and the changes on managed resources are reflected directly on the corresponding resource in the provider. You can see all available managed resources running this command ```kubectl get crds | grep aws | sort``` and the API specification in the [documentation](https://doc.crds.dev/github.com/crossplane/provider-aws).


### Creating a S3 Bucket

Now, we are going to create a Bucket and test that is working.

- Create the MR for the bucket:
    
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

- Check that the bucket has been created in localstack:
    
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