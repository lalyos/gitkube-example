This repo represents a simple k8s application, and demonstrates a ci/cd workflow:

- perform local changes (edit index.html)
- git push to <gitkube-remote>
- a new images is build on the k8s side (gitkubed) with git-sha as tag
- image is pushed to docker repo
- deployment is patched with the git-sha image ref

## initial steps

create k8s side CRD/controler/roles:
```
kubectl create -f https://storage.googleapis.com/gitkube/gitkube-setup-stable.yaml
```

## Get ssh remote address

Expose the ssh endpoint as NodePort (i have GKE with LB disabled)

```
kubectl --namespace kube-system expose deployment gitkubed --type=NodePort --name=gitkubed
```

```
$ nodeIP=$(kubectl get no -o jsonpath='{.items[0].status.addresses[1].address}')
$ nodePort=$(kubectl get svc -n kube-system gitkubed -o jsonpath='{.spec.ports[0].nodePort}')

$ curl ${nodeIP}:${nodePort}
SSH-2.0-OpenSSH_7.4p1 Debian-10+deb9u4
```

## add SSH remote

The ssh remote address can be checked from:
```
kubectl get Remote -o yaml
```

But sometimes you have to do it manually as `ssh://<namespace>-<remote-name>@<any-node-ip>:<node-port>/~/git/<namespace>-<remote-name>`

## Docker repo authorization

`gitkube` will read your local config file `~/.docker/config.json`, and store it in a k8s secret.
I run in to various issues:
- Docker4Mac doesn't uses tht file anymore, i think it stores the credentials in the os Keychain.
- For google registries (gcr.io or eu.gcr.io) the cli was trying to get usern/passw 
  but at the end it has created a dockerconf.json which wa rejected by google.
- I've used `docker login` on an empty linux machine to generate a valid dockerconfig.json.
  - for docker.io: `docker login` and stdin used for user/pwd
  - for google : ` echo key.json| docker login -u _json_key --password-stdin https://eu.gcr.io`
    see: https://cloud.google.com/container-registry/docs/advanced-authentication#json_key_file


## Updating the secret:

So once you have a valid dockerconfig.json you can update the secret:
```
kubectl create secret generic <remotename>-regsecret \
  --from-file=.dockerconfigjson \
  --dry-run -o yaml \
  | kubectl apply -f -
```

That will update only the secret. To get it effective, you have to trigger the generation of the ConfigMap `gitkube-ci-conf` in the kube-system napespace. Which acts as a DB for all repositories.

Checking the cm:
```
kubectl get cm -n kube-system gitkube-ci-conf -o jsonpath="{.data['remotes\.json']}" | jq .
```

Regenerating the `Remote`:
```
kubectl delete Remote <remotename>
kubectl apply -f remote.yaml
```

## debugging

To see a more detailes at git push time, simple run the receiver hook in debug mode:
```
kubectl exec -it  -n kube-system $(kubectl get po -n kube-system -l app=gitkubed -o jsonpath='{.items[0].metadata.name}') -- sed -i '2 i\set -x ' /home/play-webgcr/git/play-webgcr/hooks/pre-receive
```

You can exec into the container, which receives the git push on the k8s side:
```
$ kubectl exec -it  \
   -n kube-system \
   $(kubectl get po -n kube-system -l app=gitkubed -o jsonpath='{.items[0].metadata.name}') \
   -- bash

# check the files
$ cd /home/yourproject

## check the persisted repo auth file:
$ jq . /home/<namespace-remote>/.docker/config.json 
```

checking gitkubed's logs:
```
kubectl logs \
  -n kube-system \
  $(kubectl get po -n kube-system -l app=gitkubed -o jsonpath='{.items[0].metadata.name}')
```

see the controller's log:
```
kubectl logs \
  -n kube-system \
  $(kubectl get po -n kube-system -l app=gitkube-controller -o jsonpath='{.items[0].metadata.name}') 
```

sometimes ypu just need a new commit to be able to test the process:
```
git commit --allow-empty -m "empty"
```