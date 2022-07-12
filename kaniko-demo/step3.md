# A dangerous way

Docker Daemon is running on the nodes of K8S cluster. Docker CLI sends orders tu Docker Daemon trhough a Socket. So if we mount the socket inside the docker container, we should be able to build.
This technique is called DinD (Docker in Docker)

First we create a pod running docker, and we add a sleep command to have time to enter it :
```sh
cat << EOF > docker-ind.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: docker
spec:
  containers:
  - name: docker
    image: docker
    args: ["sleep", "10000"]
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-socket
  restartPolicy: Never
  volumes:
  - name: docker-socket
    hostPath:
      path: /var/run/docker.sock
EOF
```{{execute}}

and run it on K8S :

`kubectl apply -f docker-ind.yaml`{{execute}}

We wait for the pod to be up and running
`kubectl wait --for condition=containersready pod docker`{{execute}}

Then, we exevute a shell into the image
`kubectl exec -ti docker -- sh`{{execute}}

and we try to build our image inside the containers
```sh
cd /tmp
cat << EOF > Dockerfile
FROM alpine

ENV VERSION="v1.24.1" 
RUN wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz && \
  tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin && \
  rm -f crictl-$VERSION-linux-amd64.tar.gz
CMD ["/bin/echo", "It is alive !!!"]
EOF
docker build -t my-super-image .
docker run -ti my-super-image
```{{execute}}

This is working as exepected ! Problem solved ... ?

Not quite.
First, docker daemon will soon be removed from K8S distributions.
Then, it is a major security threat : accessing docker daemon from within a container could lead to messy stuff.


Want to see it by yourself ? A pod is running a container quoting the sitcom *Friends*
You can display its logs in a second tab :
`sleep 1; kubectl logs -f friends`{{execute T2}}

Go back to the first tab. You can find the running container by querying the Docker Daemon, through the socket :
`docker run -v /run/containerd/containerd.sock:/run/containerd/containerd.sock:ro -e IMAGE_SERVICE_ENDPOINT=unix:///run/containerd/containerd.sock -e CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock -t my-super-image /bin/sh -c 'crictl ps --name friends'`{{execute T1}}

You can even kill this container :
`docker run -v /run/containerd/containerd.sock:/run/containerd/containerd.sock:ro -e IMAGE_SERVICE_ENDPOINT=unix:///run/containerd/containerd.sock -e CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock -t my-super-image /bin/sh -c 'crictl rm -f $(crictl ps --name friends -q)'`{{execute T1}}

Check pod status :
`kubectl get pods`{{execute T2}}

Not that great...

You can close Terminal 2, exit the container (type `exit`) and clean it :
```sh
kubectl delete -f docker-ind.yaml --force=true
```{{execute}}
