# How to deploy Quasar in Kubernetes with ssr

## Build the app

```sh
quasar build -m ssr
```

## Customize

1. Clone this repo.
2. Open the cloned repo with your editor, and make a 'search and replace' to change all occurrences of `myquasarapp` to the name of your app.
3. Open the files in the `k8s` directory and search for the comments starting with the word `@custom` and modify the values below them according to your needs.

## Persistent volume claim

```sh
kubectl apply -f k8s/pvc.yaml
```

(A persistent volume will be created automatically if you have 'dynamic volume provisioning')

### Copy/clone the app to the created persistent volume

One way of achieving this is to run a temporary pod with the volume mounted, and use this pod to copy or clone the app into the volume:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: temporary-quasar-pod
spec:
  containers:
  - name: quasar
    image: node:alpine
    volumeMounts:
    - name: quasar-myquasarapp
      mountPath: /quasar
    command: ["sleep", "infinity"]
  volumes:
  - name: quasar-myquasarapp
    persistentVolumeClaim:
      claimName: quasar-myquasarapp
EOF
```

then

```sh
kubectl cp /local/path/to/myquasarapp/dist/ssr temporary-quasar-pod:/quasar/myquasarapp
kubectl exec temporary-quasar-pod -- yarn --cwd /quasar/myquasarapp
```

Or

```sh
kubectl exec temporary-quasar-pod -- \
git clone https://link/to/app-ssr-dist-repo.git /quasar/myquasarapp && \
yarn --cwd /quasar/myquasarapp && \
```

When finished you can delete the temporary pod

```sh
kubectl delete po temporary-quasar-pod
```

### App secrets

If your app needs to use secrets (ex: api connection credentials), you can create them in Kubernetes using the example below:

```sh
kubectl create secret generic quasar-myquasarapp --from-literal=secret1='mYsEcREt1' --from-literal=secret2='mYsEcREt2'
```

then uncomment the secrets in the deployment section:`spec.template.spec.containers[0].env` inside `k8s/manifest.yaml`

## TLS certificate

If you have [cert manager](https://cert-manager.io/) installed, you can generate a tls certificate with:

```sh
kubectl apply -f k8s/tls_certificate.yaml
```

then uncomment the ingress `spec.tls` section inside `k8s/manifest.yaml`

## App manifest

```sh
kubectl apply -f k8s/manifest.yaml
```

This will create an ingress, service and a deployment for the app.

Point your domain name to your Kubernetes cluster to access the app.

Alternatively, if you want to test locally, add the following to your hosts file

`/etc/hosts`

```txt
192.0.2.0 myquasarapp.net
```

Change the above to the ip of your Kubernetes cluster (or local ip of the ingress controller service), you can now access your app at `http(s)://myquasarapp.net`
