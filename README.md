---
name: Kubernetes Deployment or StatefulSet Update
author: EuryeceTelecom
description: Update a Kubernetes deployment or statefulset
tags: [deploy, kubernetes, deployment, container, statefulset]
containerImage: euryecetelecom/woodpeckerci-kubernetes
containerImageUrl: https://hub.docker.com/r/euryecetelecom/woodpeckerci-kubernetes
url: https://github.com/euryecetelecom/woodpeckerci-kubernetes
---

# Kubernetes plugin for Woodpecker-CI

This plugin allows to update a Kubernetes deployment or statefulset.


## Settings

| Setting Name              | Default               | Description
| --------------------------| --------------------- | --------------------------------------------
| `kubernetes_server`       | *none*                | Kubernetes server to target (ex: https://mykubernetes.example.com) - mandatory
| `kubernetes_token`        | *none*                | Kubernetes token to use (cf Generating secrets) - mandatory / B64 encoded
| `kubernetes_cert`         | *none*                | Kubernetes certificate to use (cf Generating secrets) / B64 encoded
| `kubernetes_user`         | `default`             | Kubernetes user to use
| `deployment`              | *none*                | Deployment(s) to update - at least 1 deployment or statefulset are mandatory
| `statefulset`             | *none*                | StatefulSet(s) to update - at least 1 deployment or statefulset are mandatory
| `namespace`               | `default`             | Deployment or StatefulSet namespace
| `repo`                    | *none*                | Repository containing the image to pull from (ex: myrepo.example.com/project/image) - mandatory
| `container`               | *none*                | Container(s) to update with the image - mandatory
| `tag`                     | *none*                | Image tag to pull from - mandatory
| `wait`                    | *none*                | Wait for update to be applied (ex: true)
| `wait_timeout`            | `30s`                 | Wait timeout
| `force`                   | *none*                | Force pull the new image, to ensure an image with the same tag is updated (ex: true)

## Usage

### Update a container from one Deployment

This pipeline will update the `my-deployment` deployment with the image tagged `CI_COMMIT_SHA`

```yaml
    deploy:
        image: euryecetelecom/woodpeckerci-kubernetes
        settings:
            kubernetes_server:
                from_secret: kubernetes_server
            kubernetes_token:
                from_secret: kubernetes_token
            kubernetes_cert:
                from_secret: kubernetes_cert
            namespace: default
            deployment: my-deployment
            repo: myorg/myrepo
            container: my-container
            tag: ${CI_COMMIT_BRANCH}
        secrets:
            - kubernetes_cert
            - kubernetes_server
            - kubernetes_token
```

### Update a container from one StatefulSet

This pipeline will update the `my-statefulset` statefulset with the image tagged `CI_COMMIT_SHA`

```yaml
    deploy:
        image: euryecetelecom/woodpeckerci-kubernetes
        settings:
            kubernetes_server:
                from_secret: kubernetes_server
            kubernetes_token:
                from_secret: kubernetes_token
            kubernetes_cert:
                from_secret: kubernetes_cert
            namespace: default
            statefulset: my-statefulset
            repo: myorg/myrepo
            container: my-container
            tag: ${CI_COMMIT_BRANCH}
        secrets:
            - kubernetes_cert
            - kubernetes_server
            - kubernetes_token
```

### Update a container from one Deployment, force rollout and wait for it

This pipeline will update the `my-deployment` deployment with the image tagged `CI_COMMIT_SHA`, force rollout and wait 300s (default is 30s) for it to be ready. This helps to ensure the next pipeline step is based on the deployed container - for automatic testing purposes for example.

```yaml
    deploy:
        image: euryecetelecom/woodpeckerci-kubernetes
        settings:
            kubernetes_server:
                from_secret: kubernetes_server
            kubernetes_token:
                from_secret: kubernetes_token
            kubernetes_cert:
                from_secret: kubernetes_cert
            namespace: default
            wait: true
            wait_timeout: 60s
            force: true
            deployment: my-deployment
            repo: myorg/myrepo
            container: my-container
            tag: ${CI_COMMIT_BRANCH}
        secrets:
            - kubernetes_cert
            - kubernetes_server
            - kubernetes_token
```

### Update a container from several Deployments

Deploying containers across several deployments, eg in a scheduler-worker setup. Make sure your container `name` in your manifest is the same for each pod.
    
```yaml
    deploy:
        image: euryecetelecom/woodpeckerci-kubernetes
        settings:
            kubernetes_server:
                from_secret: kubernetes_server
            kubernetes_token:
                from_secret: kubernetes_token
            kubernetes_cert:
                from_secret: kubernetes_cert
            namespace: default
            deployment: [server-deploy, worker-deploy]
            repo: myorg/myrepo
            container: my-container
            tag: ${CI_COMMIT_BRANCH}
        secrets:
            - kubernetes_cert
            - kubernetes_server
            - kubernetes_token
```

### Update multiple container from a Deployment

Deploying multiple containers within the same deployment.

```yaml
    deploy:
        image: euryecetelecom/woodpeckerci-kubernetes
        settings:
            kubernetes_server:
                from_secret: kubernetes_server
            kubernetes_token:
                from_secret: kubernetes_token
            kubernetes_cert:
                from_secret: kubernetes_cert
            namespace: default
            deployment: my-deployment
            repo: myorg/myrepo
            container: [container1, container2]
            tag: ${CI_COMMIT_BRANCH}
        secrets:
            - kubernetes_cert
            - kubernetes_server
            - kubernetes_token
```

### TODO: To be tested - multiple containers from multiple deployments

## Required secrets

```bash
    woodpecker-cli secret add --image=euryecetelecom/woodpeckerci-kubernetes \
        --repository your-org/your-repo \
        --name KUBERNETES_SERVER \
        --value https://mykubernetesapiserver

    woodpecker-cli secret add --image=euryecetelecom/woodpeckerci-kubernetes \
        --repository your-org/your-repo \
        --name KUBERNETES_CERT \
        --value <base64 encoded CA.crt>

    woodpecker-cli secret add --image=euryecetelecom/woodpeckerci-kubernetes \
        --repository your-org/your-repo \
        --name KUBERNETES_TOKEN \
        --value eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJ...
```

When using TLS Verification, ensure Server Certificate used by kubernetes API server 
is signed for SERVER url ( could be a reason for failures if using aliases of kubernetes cluster )

## Generating secrets - RBAC

When using a version of kubernetes with RBAC (role-based access control)
enabled, you will not be able to use the default service account, since it does
not have access to update deployments.  Instead, you will need to create a
custom service account with the appropriate permissions (`Role` and `RoleBinding`, or `ClusterRole` and `ClusterRoleBinding` if you need access across namespaces using the same service account).

As an example (for the `default` namespace):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deploy
  namespace: default
automountServiceAccountToken: true

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deploy
  namespace: default
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get","list","patch","update", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deploy
  namespace: default
subjects:
  - kind: ServiceAccount
    name: cicd-deploy
    namespace: default
roleRef:
  kind: Role
  name: cicd-deploy
  apiGroup: rbac.authorization.k8s.io

---

apiVersion: v1
kind: Secret
metadata:
  name: cicd-deploy-secret
  namespace: default
  annotations:
    kubernetes.io/service-account.name: cicd-deploy
type: kubernetes.io/service-account-token

```

Once the service account is created, you can extract the `ca.cert` and `token`
parameters as mentioned for the default service account above:

```
kubectl -n default get secret/cicd-deploy-secret -o yaml | egrep 'ca.crt:|token:'
```

## Improvements / Ideas
Replace the current kubectl bash script with a go implementation.

