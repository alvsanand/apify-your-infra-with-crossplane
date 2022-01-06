---
hide:
  - navigation
---
# Integrate in GitOps

GitOps is a set of practices to manage infrastructure and application configurations using Git, an open source version control system. GitOps works by using Git as a single source of truth for declarative infrastructure and applications. The Git repository contains the entire state of the system so that the trail of changes to the system state are visible and auditable.

Using GitOps tool such as Argo CD or Flux CD, we will be able to make our target system match the desired state that is coded in the Git repository. So when we will deploy new resources or update an existing ones, after updating the repository the automated process will apply the changes.

In the last laboratory, we will apply this concepts to our infrastructure using Crossplane and [Argo CD](https://argo-cd.readthedocs.io/en/stable/).

!!! note
    If you want to know more about Gitops and Argo CD, please read my article [From GIT to Kubernetes in 10 minutes with ArgoCD](https://santanderglobaltech.com/en/from-git-to-kubernetes-in-10-minutes-with-argocd/).

## Creating the repository

As said before a Git will be the single source of truth for our infrastructure and will contain the entire state and history of it so that the trail of changes to the system state are visible and auditable.

### Create a Github repository

First thing to do is to create a Git repository that we will use to store the Crosplanes files.

```bash
gh repo create --public acw-crossplane-with-argocd --clone --gitignore Python

cd acw-crossplane-with-argocd
```

### Add Crossplane files

Now, we have to create the Crosplanes files but for now we will not commit them.

- Create a ```CompositeResourceDefinition``` file:
    
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

- Create a ```Composition``` file:
    
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

- Create a ```Claim``` file:
    
    ```bash
    cat > claim.yaml <<EOF
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
                              fmt: "%s-acw"
    EOF
    ```

## Configure ArgoCD

Now that we have a repository, we have to create an [Application](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications) in ArgoCD so Crossplane resources will be deployed automatically.

- Create a new namespace for the app.

    ```bash
    kubectl create namespace another-crossplane-workshop
    ```

- Obtain HTTPS url of the GIT repository

    ```bash
    HTTPS_REPO_URL=$(git remote show origin |  sed -nr 's/.+Fetch URL: git@(.+):(.+).git/https:\/\/\1\/\2.git/p')
    ```

- Create a new Application in auto mode and listening to the master:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: acw-storage
      namespace: another-crossplane-workshop
    spec:
      destination:
        namespace: another-crossplane-workshop
        server: https://kubernetes.default.svc
      project: default
      source:
        repoURL: $HTTPS_REPO_URL
        path: .
        targetRevision: master
      syncPolicy:
        automated: {}
    EOF
    ```

## Commit changes

After ArgoCD is ready to watch for changes, we will push the files to the repo in order to force the deployment.

- Create a new namespace for the app.

    ```bash
    kubectl create namespace another-crossplane-workshop
    ```

- Obtain HTTPS url of the GIT repository

    ```bash
    HTTPS_REPO_URL=$(git remote show origin |  sed -nr 's/.+Fetch URL: git@(.+):(.+).git/https:\/\/\1\/\2.git/p')
    ```

- Create a new Application in auto mode and listening to the master:

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: acw-storage
      namespace: another-crossplane-workshop
    spec:
      destination:
        namespace: another-crossplane-workshop
        server: https://kubernetes.default.svc
      project: default
      source:
        repoURL: $HTTPS_REPO_URL
        path: .
        targetRevision: master
      syncPolicy:
        automated: {}
    EOF
    ```