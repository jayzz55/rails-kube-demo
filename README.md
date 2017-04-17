# Rails on docker-compose and Kubernetes (GKE)

`docker-compose up`

This is a demo app for showcasing a Rails 5 application running on docker-compose in development and Kubernetes on GKE in production.


Read Part I: Rails on docker-compose [here](https://medium.com/@jbielick/rails-on-docker-compose-7e2cf235fa0e).

Read Part II: Rails on Kubernetes [here](https://medium.com/@jbielick/rails-on-kubernetes-8cd4940eacbe)

## Steps:

### Setting up:
* Install [Google Cloud SDK](https://cloud.google.com/sdk/)
* install kubectl:
```
gcloud components install kubectl
```
* Configure your PROJECT_ID like so:
```
gcloud config set project PROJECT_ID
```
* Set the specific region:
```
gcloud config set compute/zone {your favorite zone here}
```

### Creating the cluster:
* Create a cluster:
```
gcloud container clusters create rails \
    --enable-cloud-logging \
    --enable-cloud-monitoring \
    --machine-type n1-standard-2 \
    --num-nodes 1
```
This command creates a cluster of n1-standard-2 type VMs with monitoring and logging enabled. The name of the cluster will be “rails”. The command should hang until the cluster creation is complete.
* provide the credentials to kubectl where “rails” is the name of your cluster
```
gcloud container clusters get-credentials rails
```
*If you get an authentication or permissions error, try running `gcloud auth login` and allow the permissions requested.*
* To check the status of the cluster:
```
kubectl cluster-info
```

### Setup a pool of nodes in GKE that only our database container is allowed to use:
* Set up the DB node pools:
```
gcloud container node-pools create db-pool \
    --machine-type=n1-highmem-2 \
    --num-nodes=1 \
    --cluster=rails
```

### Creating Secrets
* `kubectl create -f kube/app-secrets.yml` to create the secret object in kubernetes master
* You can edit the file later with `kubectl edit secret/app-secrets`

### Creating a persistent disk
* Use the following command to create a persistent disk with the name db-data and 300GB of storage in GCE (you can resize this later):
```
gcloud compute disks create --size 300GB --type pd-ssd db-data
```

### Deploying the persistent disk

#### Creating a persistent disk
* Use the following command to create a persistent disk with the name db-data and 300GB of storage in GCE (you can resize this later):
```
gcloud compute disks create --size 300GB --type pd-ssd db-data
```

#### Creating the MySQL deployment
* You can now create the mysql deployment with:
```
kubectl create -f kube/mysql-deployment.yml
```
* Run `kubectl get pods` and you should see your pod running after a minute or so

#### Creating the MySQL service
* To create service: `kubectl create -f kube/mysql-service.yml`
* To list services: `kubectl get services`
* **if you’d like to connect to your mysql pod from your local machine, try `kubectl port-forward [pod name] 3306:3306` and connect to localhost:3306 with a mysql client. The pod name will be available via `kubectl get pods`. **

### Deploying the application

### Pushing the container to the registry
* Tag an existing container named `app` with v1: `docker tag app us.gcr.io/rails-kube-demo/app:v1`. This follows the format `us.gcr.io/$PROJECT_ID/$CONTAINER_NAME:$TAG`, where the TAG can be git commit SHA, but we use v1 in this case here.
* you can push it to the google cloud registry with `gcloud docker push us.gcr.io/rails-kube-demo/app:v1`

#### Creating the web deployment
* You can now create the mysql deployment with:
```
kubectl create -f kube/web-deployment.yml
```
* Run `kubectl get pods` and you should see your pod running after a minute or so

##### Troubleshooting
* use `kubectl describe pod [pod name]` to check the status of that pod deployment
* you can use `kubectl apply -f kube/web-deployment.yml` to apply changes that you’ve made to your spec
* If your container is having trouble starting, you can use `kubectl exec -it [pod name] bash` to open an interactive bash shell

### Exposing the web pods
* We can reserve a static IP from google cloud with the following command: `gcloud compute addresses create app-external — region=us-central1`
* Use it for the `loadBalancerIP` directive in kube/web-service.yml
* Use `kubectl create -f kube/web-service.yml` to create the load balancer.

### Scaling up
* To scale up (eg: to 10 pods): `kubectl scale deployments/web --replicas 10`
Kubernetes will schedule those pods on nodes in the cluster. When it runs out of room on those nodes, the status will be Pending.
* To see details about what may have gone wrong scaling up, you can see the details of a pod with `kubectl describe pod [pod name]`

### Autoscaling
* Use [this spec](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) to autoscale, which continuously checks the metrics of our choice and determines if more or fewer pods need to be running
* Create this with `kubectl create -f kube/web-autoscaler.yml` and you’re good to go

### Deploying new code
After your container is built, tag the container like so: `docker tag app us.gcr.io/rails-kube-demo/app:v2` and push it to the google registry with `gcloud docker push us.gcr.io/rails-kube-demo/app:v2`. Remember when we used that image name and tag in the web-deployment.yml? Open that file back up and change the `image:` value to the image name you just pushed. In our case, we’ll change the `:v1` to `:v2`. Use `kubectl apply -f kube/web-deployment.yml` and kubernetes will automatically start rolling out your new container.
* use `watch -n1 'kubectl get pods'` to watch the rollout occur.

### Deploy tasks
1. Build a new container of new code.
2. Compile assets inside the container—prepare the container to be deployed.
3. Put one of these new containers in the cluster and run rake db:migrate and any other deploy tasks inside that one container.
4. Rollout the new containers to replace all existing app containers.

* Create a file like the following in a folder named script in your project root (script/deploy-tasks.sh)
* Make that file executable with `chmod +x script/deploy-tasks.sh`

When we deploy, we’ll probably automate the creation of this kubernetes job and the container `image:` will be different each time. For that reason, we’re going to use `envsubst` and an ENV variable to make this job spec reusable.

### Deployment
So start to finish, the deploy process to build a container of the existing code and roll it out (with deploy-tasks) is as follows:
```
docker build -t app .
docker tag app us.gcr.io/rails-kube-demo/app:v2
gcloud docker push us.gcr.io/rails-kube-demo/app:v2
script/deploy.sh v2
```
