# How to deploy Quasar in Kubernetes with ssr

## Build the app

```sh
quasar build -m ssr
```

## Customize

1. Clone this repo.
2. Open the cloned repo with your editor, and make a 'search and replace' to change all occurrences of `myquasarapp` to the name of your app, and all occurrences of `example.net` to your domain name.
3. Open the files in the `k8s` directory and search for the comments starting with the word `@custom` and modify the values below them according to your needs.

## Persistent volume claim

```sh
kubectl apply -f k8s/pvc.yaml
```

A persistent volume will be created automatically if you have 'dynamic volume provisioning', if not then create the volume manually.

### Copy/clone the app to the created persistent volume

One way of achieving this is to run a temporary pod with the volume mounted, and use this pod to copy or clone the app into the volume:

```sh
kubectl apply -f k8s/temporary-git-pod.yaml
kubectl apply -f k8s/temporary-nodejs-pod.yaml
```

We will make two copies of the app (blue and green), one will be active, and the other will be used as a staging copy to test new updates/features of the app, if our testing goes well, we flip a switch to swap the roles of the app copies and expose the new features to our customers.

Read more about blue/green deployments concept [here](https://www.haproxy.com/blog/rolling-updates-and-blue-green-deployments-with-kubernetes-and-haproxy/#upgrade-using-a-blue-green-deployment)

#### Create the app blue copy

(you might need to wait a little to get the temporary pod docker image pulled, check with `kubectl get po temporary-git-pod`)

```sh
kubectl cp /local/path/to/myquasarapp/dist/ssr temporary-git-pod:myquasarapp-blue
```

Or

```sh
kubectl exec temporary-git-pod -- git clone https://link/to/app-ssr-dist-repo.git myquasarapp-blue
```

#### Install the dependencies and create the green app copy

```sh
kubectl exec temporary-nodejs-pod -- yarn --cwd myquasarapp-blue
kubectl exec temporary-nodejs-pod -- cp -r myquasarapp-blue myquasarapp-green
```

When finished you may delete the temporary pods

```sh
kubectl delete -f k8s/temporary-git-pod.yaml
kubectl delete -f k8s/temporary-nodejs-pod.yaml
```

### App secrets

If your app needs to use secrets (ex: api connection credentials), you can create them in Kubernetes using the example below:

```sh
kubectl create secret generic quasar-myquasarapp --from-literal=secret1='mYsEcREt1' --from-literal=secret2='mYsEcREt2'
```

then uncomment the secrets in the deployment section:`spec.template.spec.containers[0].env` inside `k8s/manifest-*.yaml`

## TLS certificate

If you have [cert manager](https://cert-manager.io/) installed, you can generate a tls certificate with:

```sh
kubectl apply -f k8s/tls_certificate.yaml
```

then uncomment the `spec.tls` section inside `k8s/switch-to-*.yaml`

## App manifest

This setup assumes you have an ingress controller installed in your Kubernetes cluster. Adjust the code as needed if you don't use an ingress controller.

```sh
kubectl apply -f k8s/manifest-blue.yaml
kubectl apply -f k8s/manifest-green.yaml
```

This will create a service and a deployment for each copy of the app.

## App ingress

```sh
kubectl apply -f k8s/switch-to-blue.yaml
```

This will create an ingress that points `example.net` to the blue copy (active for now), and `stg.example.net` to the green copy (staging for now).

Point your domain name to your Kubernetes cluster to access the app.

You can also access the app copies locally by adding the following to your hosts file

`/etc/hosts`

```txt
192.0.2.0 example.net
192.0.2.0 stg.example.net
```

Change the above ip to the ip of your Kubernetes cluster (or local ip of the ingress controller service).

You can now access the active copy of your app at `http(s)://example.net`, and the staging copy at `http(s)://stg.example.net`

## Handling the app updates

Let's say now we made some changes to the app and we want them deployed to production, the necessary step before deploying to production is to test first that our new code is still running fine in the production environment.

Let's update the green copy with the changes:

(If you have deleted the temporary pods, start them again)

```sh
kubectl exec temporary-git-pod -- git -C myquasarapp-green pull
kubectl exec temporary-nodejs-pod -- yarn --cwd myquasarapp-green
kubectl rollout restart deployment quasar-myquasarapp-green
```

(`yarn` command is needed only if the app "dependencies" have changed)

Now test the staging app, and if all looks fine make the swap:

```sh
kubectl apply -f k8s/switch-to-green.yaml
```

This command will patch the existing ingress, resulting in `example.net` pointing to the updated copy (green) and `stg.example.net` pointing to the blue copy.

On the next app update we do the same but with swapped colors:

```sh
kubectl exec temporary-git-pod -- git -C myquasarapp-blue pull
kubectl exec temporary-nodejs-pod -- yarn --cwd myquasarapp-blue
kubectl rollout restart deployment quasar-myquasarapp-blue
```

Test, then:

```sh
kubectl apply -f k8s/switch-to-blue.yaml
```

## Tip

You can track your built ssr bundle with git to facilitate transferring updates to your production and staging copies, however when you run `quasar build -m ssr` it will delete `.git` directory, to workaround this, do the following instead:

```sh
mv dist/ssr/.git /tmp/ && quasar build -m ssr && mv /tmp/.git dist/ssr/
```

Then git add, commit and push to remote.
