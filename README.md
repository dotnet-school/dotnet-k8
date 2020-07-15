Resources: 

-  https://learnk8s.io/nodejs-kubernetes-guide
- https://v1-15.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/
- https://learnk8s.io/autoscaling-apps-kubernetes



### Setup project

- clone this repo  https://github.com/dotnet-school/dotnet-docker

- push your docker image and make sure it is public (we will see how to handle private one later)

  ```bash
  # We will use this image with kubernetes
  cd TodoApp
  docker build -t nishants/todo_app:v0.1 .
  docker push nishants/todo_app:v0.1
  ```

  

- **Kubectl**
  
  - install kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl/
  - Kubernetes CLI
  - To interact with kubernetes clusture
  - Clusture could be managed, hosted or based on minikube
- **Minikube** 

  - install minikube : https://kubernetes.io/docs/tasks/tools/install-minikube/

  - Creates a singe node k8 clusture on a local machine.

  - It uses a VM (e.g. virtual box). 

  - Ideal for development.

    

### Setup a local K8 cluster with Minikube

- Start a single node local clusture using minikube

  ```bash
  minikube start
  
  # minikube v1.11.0 on Darwin 10.15.3
  # Automatically selected the hyperkit driver
  # Starting control plane node minikube in cluster minikube
  # Creating hyperkit VM (CPUs=2, Memory=2200MB, Disk=20000MB) ...
  # Preparing Kubernetes v1.18.3 on Docker 19.03.8 ...
  # Verifying Kubernetes components...
  # Enabled addons: default-storageclass, storage-provisioner
  # Done! kubectl is now configured to use "minikube"
  ```

  - this may take a few minutes 
  - it starts the vitual machine (virtual box)
  - and prepares a local k8 clusture with single node (one vm instance)

- Check the created cluster : 

  ```bash
  kubectl cluster-info
  # Kubernetes master is running at https://192.168.64.2:8443
  # KubeDNS is running at https://192.168.64.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
  # To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
  ```

- Our k8 cluster is ready now.



### Understanding Kubernetes Configuration required for production setup

- **How K8 works ?**
  - It uses declarative style,
  - i.e. **we tell what is the desired state for a cluster using YAML declaration**
  - and run the `kubectl apply`
  - Kubernetes automactically makes changes to the cluster based on our desired state.

- **Our desired state in of cluster requires**
  - running app as a docker container
  - running mongo as a docker container
  - persistence disk space where mongo can write its data
  - load balancer to receive traffic from internet (from outside the cluster)
  - ability for app to talk to mongo 
- **In Kubernetes language what we need is** 
  - **Deployment**
    - **todo-app** : run todo app container 
    - **mongo-db** : run mongo containers
  - **Service**
    -  **todo-app-service** : routing external traffic to our app container
    - **mongo-service**: to route internal traffic to mongo containers
  - **Persistence Volume Claim**
    - **mongo-disk:** Persistence disk space where mongo can write its data



### Our requirements in Kubernete's language

| What we need in our cluster ?                                | What Kubernetes resource we need ?               |
| ------------------------------------------------------------ | ------------------------------------------------ |
| Running app as a docker container                            | **Deployment** resource for app's docker image   |
| Running mongo as a docker container                          | **Deployment** resource for mongo's docker image |
| Persistent disk space where mongo can write its data         | **Persistance Volume Claim**                     |
| Load balancer to receive traffic from internet (outside the cluster) | **Service** of type load balancer                |
| Ability for app to talk to mongo                             | **Service** for the mongo                        |



### Kubernetes API versions

- When talking to Kubernetes, we alwasy define what API language are we talking in
- Managed Kubernetes like Azure Kubernetes will usually not have the latest API's
- When referring to docs, make sure you are looking at correct api versions



### Declaring our Kubernetes configuration

- We can declare all configuration in a single file, but we will use two files. One for mongo and other for app.

- We will put both our yaml files in a single folder `k8` and run `kubectl apply ` on this folder

  ```bash
  # Create k8 directory and files
  mkdir k8
  cd k8
  touch todo-app.yml
  touch mongo-db.yml
  ```

- Create a deployment declaration for todo app `todo-app.yml`

  ```yaml
  # This part creates a pod that runs our docker image
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: todo-app
  spec:
    # we only want one replica of our container for now
    replicas: 1  
    selector:
      matchLabels:
        app: todo-app
    template:
      metadata:
        labels:
          app: todo-app
      spec:
        containers:
          - name: todo-app
            image: nishants/todo_app:v0.1   # Our docker image on docker hub
            ports:
              - containerPort: 80           # Port that our app listens to
            env:
              - name: TODO_APP_MongoSettings__ConnectionString
                value: mongodb://mongo-service:27017/dev # mongo-service that exposes mongo db to us
            imagePullPolicy: Always
  ```

- Apply kubernetes configuration to our local cluster : 

  ```bash
  # apply all yaml config from k8 folder
  kubectl apply -f k8
  ```

- This will create a pod that runs our docker container. But we cannot access this from outside the cluster.



- To access the app, we will create the load balancer resource in same file as : 

  ```yaml
  # This part creates a load balancer pod that receives traffic from
  # internet and load-balances to our pods
  apiVersion: v1
  kind: Service
  metadata:
    name: todo-app-service
  spec:
    selector:
      app: todo-app     # This makes load balancer point to todo-app deployment
    ports:
      - port: 80        
        targetPort: 80  # The port our container(in pods) listens to
    type: LoadBalancer
  ---
  # This part creates a pod that runs our docker image
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: todo-app
  spec:
    # we only want one replica of our container for now
    replicas: 1  
    selector:
      matchLabels:
        app: todo-app
    template:
      metadata:
        labels:
          app: todo-app
      spec:
        containers:
          - name: todo-app
            image: nishants/todo_app:v0.1   # Our docker image on docker hub
            ports:
              - containerPort: 80           # Port that our app listens to
            env:
              - name: TODO_APP_MongoSettings__ConnectionString
                value: mongodb://mongo-service:27017/dev # mongo-service that exposes mongo db to us
            imagePullPolicy: Always
  ```

  

- Now run the apply command again, 

  ```bash
  # Apply all yaml config from k8 folder
  kubectl apply -f k8
  
  # Get the url for todo-app-service
  minikube service todo-app-service 
  # |-----------|------------------|-------------|---------------------------|
  # | NAMESPACE |       NAME       | TARGET PORT |            URL            |
  # |-----------|------------------|-------------|---------------------------|
  # | default   | todo-app-service |          80 | http://192.168.64.5:31019 |
  # |-----------|------------------|-------------|---------------------------|
  ```

  

- Open the url in browser, we will see our app 

  ![image-20200715155401215](./docs/images/disabled-app.png)

- **Out app is disbaled, as it has no connection with database.** 

- To make sure, lets look into the logs of the pod : 

  ```bash
  kubectl get pods
  # NAME                        READY   STATUS    RESTARTS   AGE
  # todo-app-7cf78bcc8d-njcmn   1/1     Running   0          48m
  
  kubectl logs todo-app-7cf78bcc8d-njcmn
  
  # An unhandled exception has occurred while executing the request.
  # System.TimeoutException: A timeout occured after 30000ms selecting a server using CompositeServerSelector{ Selectors = MongoDB.Driver.MongoClient+AreSessionsSupportedServerSelector, LatencyLimitingServerSelector{ AllowedLatencyRange = 00:00:00.0150000 } }. Client view of cluster state is { ClusterId : "1", ConnectionMode : "Automatic", Type : "Unknown", State : "Disconnected", Servers : [{ ServerId: "{ ClusterId : 1, EndPoint : "Unspecified/mongo-service:27017" }", EndPoint: "Unspecified/mongo-service:27017", State: "Disconnected", Type: "Unknown", HeartbeatException: "MongoDB.Driver.MongoConnectionException: An exception occurred while opening a connection to the server.
  ```

  - So the app had error trying to connect with mongo db as expected



### Setup Mongo DB in our cluster

- Create a peristence volume that mongo db can use to save data : 

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: mongo-disk
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 256Mi
  ```

  





Create a Deployment Resource with 

- Resource - Deployment, PersistentVolumeClaim, Service, 



# Questoins

- how do we scale disk ?
- how do we scale mong ? (CPU ?)
- how do we scale app ?
- Can we scale app based on some data in mongo ?



###Some useful commands 

```bash
## Operatoins ##########
kubectl apply -f dir-name # apply configurations from a directory
kubectl scale --replicas=2 deployment/knote # Scale deployed containers

## Minicube ##########
minikube start
minikube stop
minikube logs
minikube status
minikube service <name>    # get service/open url for load balancer service

## Pods ##########
kubectl get pods            # show all pods
kubectl get pods --watch    # watch all pods
kubectl describe pod <name> # show status of a pod
kubectl logs <pod-name>     # view stdout of a pod
kubectl get pods -l app=todo-app --watch

## Docs ##########
kubectl explain <resource> 		# docs of a resource
kubectl explain deployment 		# docs of deployment resource
kubectl explain <resource>.<path>.<path> # describe yaml field 
kubectl explain deployment.spec.template.spec.containers # describe containers field of deployment
```