# Developing Inside Minikube


## Problem
Recently I have the need to write code that interacts with kubernetes outside and inside the cluster. Outside is easy I can just execute my application on the host and it sends the requests off to the API. Developing insight would be a pain if everytime I made a change to a line of code I would have to rebuild the container image and redeploy the pod using that new image. Luckily minikube lets you mount directories into the master node.

{{< figure src="https://media.giphy.com/media/a5viI92PAF89q/giphy.gif"  >}}

## Solution
So, I need to mount a directory into minikube, create a pod that then mounts that directory inside of it. The pod should also not execute the code as that might cause it to crash but stay alive to allow me to `exec` in.

### Mounting a directory into minikube
[Minikube](https://minikube.sigs.k8s.io) allows mounting a volume into its single node cluster. To achieve this you toggle a flag when starting the cluster.

```shell
minikube start --mount=true
```

This by default mounts your `/Home` directory on your host into the node at `/minikube-host`. This is probably a bad idea if you have a large amount of data in your home directories. This can be solved by passing a second flag to minikube.

```shell
minikube start --mount=true --mount-string=$HOME/go/src/project:/data
```

Here I mount my golang project under my home directory into `/data` on the node. I can check this worked via `ssh`'ing into the node.

```shell
minikube ssh
ls /data
main.go
```

#### Using minikube mount
Minikube also has the `mount` command which would look like this.
```shell
# start minikube
minikube start

# mount my project into the node
minikube mount $HOME/go/src/project:/data
```
The problem with this is it runs as a process and must stay running for it to work.
> 📌  NOTE: This process must stay alive for the mount to be accessible ...

### Mounting the directory into a pod
Now I have access to the data in the kube node I just have to mount it into the pod. Luckily kubernetes supports this with a `hostPath` volume type.
{{< admonition type=warning title="hostPath" open=true >}}
Avoid using `hostPath` in a production setting. They can drastically reduce the isolation provided between a container/pod and the host.
{{< /admonition >}}
I need to declare a volume as part of my pod spec. I can create a `hostPath` volume that takes my `/data` directory and I'll call it `host-volume`
```yaml
  volumes:
  - name: host-volume
    hostPath:
      path: /data
```

Next in my in a container I can use `volumeMount` I can mount the volume I just declared into the container.

```yaml
volumeMounts:
    - mountPath: /go/src
      name: host-volume
```
Finally for my container I will issue a command that will act as a small `init` process effectively keeping the pod alive allowing me to `exec` in and execute my code.
```yaml
    command: ["sh", "-c", "trap 'echo Bye' SIGTERM SIGINT; sleep infinity & wait"]
```
---
The complete pod manifiest is here:
```yml
apiVersion: v1
kind: Pod
metadata:
  name: development
spec:
  containers:
  - name: development
    image: golang:alpine
    volumeMounts:
    - mountPath: /go/src
      name: host-volume
    command: ["sh", "-c", "trap 'echo Bye' SIGTERM SIGINT; sleep infinity & wait"]
    
  volumes:
  - name: host-volume
    hostPath:
      path: /data
  restartPolicy: OnFailure
```

Now I can create the pod
```bash
kubectl apply -f development-pod.yaml
```
`exec` in and run my code
```bash
kubectl exec -it development -- sh
# cd into project under go/src
cd src/project
go run main.go
```

{{< figure src="https://media.giphy.com/media/XreQmk7ETCak0/giphy.gif"  >}}
Now I can write code on my host and execute it freely from within a pod. 🎉

### References
- https://stackoverflow.com/questions/48534980/mount-local-directory-into-pod-in-minikube
- https://kubernetes.io/docs/concepts/workloads/pods/ 
- https://kubernetes.io/docs/concepts/storage/volumes/#hostpath

