# MicroK8s

## Installing

### MicroK8s

In order to run this project you must first install MicroK8s in your host machine.
You can install it by following the instructions at the [MicroK8s official web site](https://microk8s.io/tutorials)
according to your operating system.

> In this tutorial we're running MicroK8s in a MacOS. So take everything with a grain of salt and
> add the necessary changes for your OS.

### Enabling addons

After installing MicroK8s, you must enable the necessary addons for this project to work.
In this case, only the `ingress`, which you can enable by running

```bash
microk8s enable ingress
```

### \[OPTIONAL\] Enabling Kubernetes dashboard

> Yet To be written

## Running the app

### 1. Building and pushing Docker image

In order to run the [express](https://expressjs.com/) server in your MicrK8s/Kubernetes cluster you must
first build and push a docker image of the project to [Docker Hub](https://hub.docker.com/). You can find instructions
of how to do this in [Docker Hub tutorials page](https://docs.docker.com/docker-hub/). It is necessary to have some knowledge on Docker.

In general, after creating a Docker Hub account, you must use Docker to build the current image described in the [Dockerfile](./Dockerfile)
and then push it to Docker Hub after loging in. It would be something like this:

```bash
docker login
docker build -t express-app .
docker push express-app
```

### 2. Changing MicroK8s to use the pushed image

Currently MicroK8s is configured to use my pushed Docker image (gotassinari/express-microk8s).
You must changed it for your pushed image in [deployment.yml](./deployment.yml).

For example:

```diff
public class Hello1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: express
spec:
  replicas: 3
  selector:
    matchLabels:
      app: express
  template:
    metadata:
      labels: # labels to select/identify the deployment
        app: express
    spec:     # pod spec                  
      containers: 
      - name: express 
-        image: gotassinari/express-microk8s
+        image: yourdockerhubaccount/express-microk8s
        ports:
        - containerPort: 3000
```

### 3. Running your app in MicroK8s cluster

Now you're all set to run the application in your cluster. As MicroK8s only runs
in Linux OS, MacOS (and Windows, I suppose) actually run it in a Virtual Machine.

In MacOS you can see this virtual machine with [multipass](https://multipass.run/) (which is needed)
to run MicroK8s in MacOS). Just run `multipass list` and you should see a running `microk8s-vm`.

```bash
$ multipass list
                                                                           
Name                    State             IPv4             Image
microk8s-vm             Running           192.168.64.2     Ubuntu 18.04 LTS
                                          10.1.254.64
```

> As MicroK8s is running in a VM you must know that everytime you run a MicroK8 command in your host machine
> it is actually running it in `microk8s-vm` and showing its output in your console.

Now, in order to run the app, you must create the deployment defined in [deployment.yml](./deployment.yml). Just execute the following command:

```bash
microk8s kubectl create -f ./deployment.yml
```

Now you should have your app running with 3 replicas. You can observe that running

```bash
$ microk8s kubectl get deployment

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
express               3/3     3            3           2s
```

and

```bash
microk8s kubectl get pods

NAME                                   READY   STATUS    RESTARTS       AGE
express-8c84cd9fd-sxp7k                1/1     Running   0              41s
express-8c84cd9fd-w6pcs                1/1     Running   0              41s
express-8c84cd9fd-krhx5                1/1     Running   0              41s
```

This shows that you have a deployment running with 3 pods/replicas.

### 4. Exposing the pods to your host machine under a service

Once you have your pods running, you must now expose it to your host machine using a service
and ingress. This will allow you to make requests to the express server from you host machine.

It is actually quite simple. First, you must create the service, which will act as a load balancer
for the pods inside your MicroK8s cluster. To do that, run

```bash
microk8s kubectl create -f ./service.yml
```

You should be able to see your service is running using the following command

```bash
$ microk8s kubectl get service

NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.152.183.1     <none>        443/TCP          17d
express      LoadBalancer   10.152.183.247   <pending>     3000:31349/TCP   22s
```

Now, you must expose this service to your host machine using an ingress. Again, it is quite
straightforward. Just run

```bash
microk8s kubectl create -f ./ingress.yml
```

You can see your ingress is running using the following command

```bash
$ microk8s kubectl get ingress 

NAME              CLASS    HOSTS   ADDRESS     PORTS   AGE
minimal-ingress   public   *       127.0.0.1   80      25s
```

### 5. Accessing your app from your host machine

Now you must only find the node IP in your host machine and make a request
to the express app. To to this, execute the following command

```bash
$ microk8s kubectl get node -o wide

NAME          STATUS   ROLES    AGE   VERSION                    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
microk8s-vm   Ready    <none>   17d   v1.22.3-3+9ec7c40ec93c73   192.168.64.2   <none>        Ubuntu 18.04.6 LTS   4.15.0-161-generic   containerd://1.5.2
```

The node IP will be under `INTERNAL-IP`. Now, all you ahve to do is access this IP in your web
browser. It should show a `hello-world` page just like this one:

And there it goes! Everything is up and running!