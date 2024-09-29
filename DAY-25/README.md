
---

# Kubernetes ServiceAccount API Token Guide

This guide explains how to manually create a long-lived API token for a Kubernetes ServiceAccount and how to use it for API access. It covers the following tasks:

- Creating a ServiceAccount
- Manually generating a secret for the ServiceAccount token
- Patching the ServiceAccount to use the secret
- Retrieving and using the API token for authentication
- Checking if the ServiceAccount has the required permissions
- Granting permissions if necessary

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

## 6. Verify if the ServiceAccount Can Get Pods

You can check if the ServiceAccount has permission to get pods using the following command:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:<namespace>:<service-account-name>
```

Example:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:build-robot
```

If the response is:

```bash
yes
```

It means the ServiceAccount has permission to get pods.

If the response is:

```bash
no
```

The ServiceAccount does not have the required permission, and you'll need to grant it.

## 7. Grant Permission to Get Pods

If the ServiceAccount doesn't have permission to get pods, you can create a Role and RoleBinding to grant the necessary access.

### 7.1 Create a Role

Create a YAML file for the Role with permissions to get pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

### 7.2 Create a RoleBinding

Create a YAML file for the RoleBinding to bind the Role to the ServiceAccount:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: build-robot
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply both files:

```bash
kubectl apply -f <role-file>.yaml
kubectl apply -f <rolebinding-file>.yaml
```

### 7.3 Re-check Permissions

After applying the necessary Role and RoleBinding, you can re-check if the ServiceAccount can get pods:

```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:build-robot
```

If the permission is successfully granted, you'll see:

```bash
yes
```

---

This guide provides a comprehensive way to create and manage long-lived API tokens for Kubernetes ServiceAccounts, including permission handling for API access.