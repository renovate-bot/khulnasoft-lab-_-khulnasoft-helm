# Container registry credentials

Khulnasoft Security components docker images are available in our private repository, which requires authentication. By default, the charts create a secret based on the values.yaml file.

1. Create a new namespace named `khulnasoft`:

```bash
kubectl create namespace khulnasoft
```

2. Create the image pull secret:

```bash
kubectl create secret docker-registry khulnasoft-registry-secret --docker-server="registry.khulnasoft.com" --namespace khulnasoft --docker-username="user@example.com" --docker-password="<Password>" --docker-email="user@example.com"
```

> Note: in case you using the csp chart the chart can create the image pull secret automatically.

3. After creating the secret manually, you should set the chart to not create one:

`values.yaml`:

```yaml
imageCredentials:
  create: false
```
