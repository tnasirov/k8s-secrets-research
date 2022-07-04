# k8s-secrets-research

# Problem

One of our clients is running Kubernetes on AWS (EKS + Terraform). At the moment, they store secrets like database passwords in a configuration file of the application, which is stored along with the code in Github. The resulting application pod is getting an ENV variable with the name of the environment, like staging or production, and the configuration file loads the relevant secrets for that environment.

We would like to help them improve the way they work with this kind of sensitive data.

Please also note that they have a small team and their capacity for self-hosted solutions is limited.

Provide one or two options for how would you propose them to change how they save and manage their secrets.


# Suggested Solution

Use Aws SecretsManager and/or SSM ParameterStore to keep your secrets.

Use [external-secrets](https://github.com/external-secrets/external-secrets)
for retreiving the secrets from aws managed secrets store and pass to the pod as an environment variable

For this we will need to install external-secrets controller

 Install from chart repository

```
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace \
    --set installCRDs=true
```
Conguration reference can be found here: https://external-secrets.io/v0.5.7/provider-aws-secrets-manager/


But the overall idea is:

Create an IAM role with policy that gives access to AWS secret store.

Create our secrets in AWS managed secret/config stores like Secretsmanager

```
aws secretsmanager create-secret \
     --name super-secret \
     --secret-string my-custom-secret \
     --region us-east-2
```

The first thing we need to do is define a SecretStore . This resource is where all backend-related configuration is going to be stored. To allow external-secrets to reach out to SecretManager service, we can use k8s secrets which stores AccessKey and SecretAccessKey. However more secure way will be to use IRSA. IAM role with corresponding permissions should be created with IRSA. Then we create k8s ServiceAccount:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/access-to-secrets-manager
  name: my-serviceaccount
  namespace: default
```

In the example we create a SecretStore called my-secret-store, using the resources we created from previous steps:

```
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: secretstore-sample
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-2
      auth:
        jwt:
          serviceAccountRef:
            name: my-serviceaccount
```

The next thing is to create an ExternalSecret definition. In this definition, we are going to indicate which secrets from a SecretStore we want to synchronize into a Kubernetes Secrets object. Here is an example of an ExternalSecret called my-external-secret which gets the secret we created in AWS Secrets Manager:

```
apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: my-external-secret
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: my-secret-store #The secret store name we have just created.
    kind: SecretStore
  target:
    name: my-kubernetes-secret # Secret name in k8s
  data:
  - secretKey: password # which key it's going to be stored
    remoteRef:
      key: super-secret # Our secret-name goes here
```

We can see that a secret called `my-kubernetes-secret` is created in kubernetes by the external-secrets operator.
After we can get the secret and pass it to the pod like environment variable.

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-service
spec:
  containers:
  - name: hello-service
    image: hello-service-image
    env:
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: my-kubernetes-secret
            key: super-secret
```

When secret changes K8s doesn`t reload the pod in order it to get new secret value as an environment vairable.
It can be managed by [stakater reloader](https://github.com/stakater/Reloader).
While secrets or COnfigMaps will change it will perform a rolling upgrade on relevant DeploymentConfig, Deployment, Daemonset, Statefulset and Rollout.