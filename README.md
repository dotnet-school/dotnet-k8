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
  - load balancer to receive traffic from internet (outside the cluster)
  - ability fro app to talk to mongo 
- **In Kubernetes language what we need is ** 
  - **Deployment**
    - todo-app : run todo app container s
    - mongo-db : run mongo containers
  - **Service**
    - Load balancer : routing external traffic to our app container
    - mongo-service: to route internal traffic to mongo containers
  - **Persistence Volume Claim**
    - Persistence disk space where mongo can write its data



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





### Declaring our Kubernetes configuration

- We can declare all configuration in a single file, but we will use two files. One for mongo related requirements and other for app related.

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
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: todo-app
  spec:
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
            image: nishants/todo_app
            ports:
              - containerPort: 80
            env:
              - name: MONGO_URL
                value: mongodb://mongo-service:27017/dev # we haven't created mongo service yet.
            imagePullPolicy: Always
  ```

- 

- Declaration for app : 

- 

Create a Deployment Resource with 

- Resource - Deployment, PersistentVolumeClaim, Service, 

