

# Kubernetes ServiceAccount API Token Guide

This guide explains how to manually create a long-lived API token for a Kubernetes ServiceAccount and how to use it for API access. It covers the following tasks:

- Creating a ServiceAccount
- Manually generating a secret for the ServiceAccount token
- Patching the ServiceAccount to use the secret
- Retrieving and using the API token for authentication

## 1. Create a ServiceAccount

To create a ServiceAccount, use the following command:

```bash
kubectl create sa <service-account-name>
```

Example:

```bash
kubectl create sa build-robot
```

This will create a ServiceAccount called `build-robot` in the current namespace.

## 2. Create a Secret for the ServiceAccount

Since Kubernetes 1.24 and later does not automatically generate long-lived tokens, you need to manually create a secret and bind it to the ServiceAccount. You can generate a secret using the following command:

```bash
kubectl create secret generic <secret-name> \
  --from-literal=token=$(openssl rand -base64 32) \
  --type=kubernetes.io/service-account-token \
  --namespace=<namespace>
```

Example:

```bash
kubectl create secret generic build-robot-secret \
  --from-literal=token=$(openssl rand -base64 32) \
  --type=kubernetes.io/service-account-token \
  --namespace=default
```

## 3. Patch the ServiceAccount to Use the Secret

To associate the newly created secret with the ServiceAccount, use the `kubectl patch` command:

```bash
kubectl patch serviceaccount <service-account-name> \
  -p '{"secrets": [{"name": "<secret-name>"}]}'
```

Example:

```bash
kubectl patch serviceaccount build-robot \
  -p '{"secrets": [{"name": "build-robot-secret"}]}'
```

This binds the secret to the `build-robot` ServiceAccount.

## 4. Retrieve the API Token

Once the secret is created and bound to the ServiceAccount, you can retrieve the API token using the following command:

```bash
kubectl get secret <secret-name> -o jsonpath="{.data.token}" | base64 --decode
```

Example:

```bash
kubectl get secret build-robot-secret -o jsonpath="{.data.token}" | base64 --decode
```

This will output the token, which can be used for API authentication.

## 5. Example: Using the API Token

Once you have the API token, you can use it to authenticate with the Kubernetes API. Here's an example using `curl`:

```bash
curl -k -H "Authorization: Bearer <api-token>" https://<k8s-api-server>:6443/api/v1/namespaces/default/pods
```

Replace `<api-token>` with the token you retrieved, and `<k8s-api-server>` with the URL of your Kubernetes API server.

## YAML Example: Secret Definition

Below is a YAML example to create a secret for the ServiceAccount token:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
```

Apply this YAML to manually create the secret for the ServiceAccount `build-robot`.

---

This guide provides a way to create a long-lived API token for Kubernetes ServiceAccounts, giving you the ability to authenticate and interact with the Kubernetes API securely.