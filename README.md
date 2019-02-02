# Welcome to the juorney from docker to k8s!
A juorney for ROR app from docker to k8s: It starts by dockerizing ROR app going through docker-compose, to making a kubernets deployments for that app. Here's how I did it.
## First step [[Compose](https://github.com/mareimorsy/juorney-from-ROR-to-k8s/tree/master/01-compose)]
Follow [this blog post](https://semaphoreci.com/community/tutorials/dockerizing-a-ruby-on-rails-application) to dockerize a sample Ruby On Rails app, you should end up with the following [files](https://github.com/mareimorsy/juorney-from-ROR-to-k8s/tree/master/01-compose).
## Second step [[Kompose](https://github.com/mareimorsy/juorney-from-ROR-to-k8s/tree/master/02-kompose)]
I've heard about tool called [kompose](https://github.com/kubernetes/kompose) which is used to help users who are familiar with docker-compose move to Kubernetes. kompose takes a Docker Compose file and translates it into Kubernetes resources. So I've tried it and it was a little bit disappointing because it's not as accurate as expected [here is the kompose output](https://github.com/mareimorsy/juorney-from-ROR-to-k8s/tree/master/02-kompose) anyway. Since kompose output is not acceptable for my standards, I decided to swich to plan B (the third step) which was creating the kubernetes config from scratch. 
## Third step [[Kubernetes](https://github.com/mareimorsy/juorney-from-ROR-to-k8s/tree/master/03-kubernetes)]
Since the internet is not in my favor. I'm gonna use kops to set up a cluster on AWS
```
kops create cluster \
    --ssh-public-key ./id_rsa.pub \
    --zones us-east-1a \
    --master-size t2.micro \
    --node-size t2.micro \
    --node-count 1 \
    --name ${KOPS_CLUSTER_NAME} \
    --yes
```
### Prerequisites
* Kubernetes cluster
* kubectl
* Helm (optional)
### How to run the app
1. Download/Clone the project repo `git clone https://github.com/mareimorsy/juorney-from-ROR-to-k8s.git`
2. Change directory into 03-kubernetes `cd juorney-from-ROR-to-k8s/03-kubernetes`
3. Install the app using one of the following methods
### Install the app using Helm
```
kubectl create ns drkiq
helm install ./01-postgresql/postgresql-helm -f ./01-postgresql/postgresql-helm/values.yaml --name postgresql
kubectl create -f 02-drkiq-secrets.yaml
kubectl create -f 03-db-init-job.yaml
helm install ./04-redis/redis-helm -f ./04-redis/redis-helm/values.yaml --name redis
helm install ./05-drkiq/drkiq-helm -f ./05-drkiq/drkiq-helm/values.yaml --name drkiq
```
### Install the app using kubectl
If you're not really into helm. here's how to do it ;)
```
kubectl create ns drkiq

kubectl create -f 01-postgresql/postgresql-k8s/01-postgres-svc.yaml
kubectl create -f 01-postgresql/postgresql-k8s/02-postgres-secrets.yaml 
kubectl create -f 01-postgresql/postgresql-k8s/03-postgresql.yaml     

kubectl create -f 02-drkiq-secrets.yaml

kubectl create -f 03-db-init-job.yaml

kubectl create -f 04-redis/redis-k8s/01-redis-svc.yaml
kubectl create -f 04-redis/redis-k8s/02-redis.yaml

kubectl create -f 05-drkiq/drkiq-k8s/01-drkiq-svc.yaml
kubectl create -f 05-drkiq/drkiq-k8s/02-drkiq-config.yaml
kubectl create -f 05-drkiq/drkiq-k8s/03-drkiq.yaml
```
There're many ways to view the app such as 
* exec into the pod and `curl localhost:8000`
* port forwarding `kubectl port-forward -n drkiq svc/drkiq 8000` and `curl 127.0.0.1:8000` on your local machine.
* set drkiq service type to be LoadBalancer. I really prefer that option but that would create an AWS ELB and that would cost more. We could also use a proxy in front of a service such as nginx or haproxy to manage routes, static files, caching, connections and more or using ingress.
* set drkiq service type to be NodePort, to get the service node port `kubectl get svc drkiq -n drkiq`
![Service Node Port](https://github.com/mareimorsy/juorney-from-ROR-to-k8s/raw/master/01-compose/app/assets/images/svc_node_port.png)

As you see the node port is `30423` now we need to allow that port(30423) from the ASW Security Group for nodes as following

![AWS Security groups](https://github.com/mareimorsy/juorney-from-ROR-to-k8s/raw/master/01-compose/app/assets/images/aws_security_group.png)

Now The site is ready [Click here to visit](http://3.83.78.1:30423/) !!!
### Note
* In order to stay aligned with the Twelve Factor App, The configurations are stored either in secrets or config maps and passed into the pod through deployments and environment variables.
* I've created a kubernetes job `db-init` in order to reset and migrate the DB. it must be ran before running the drkiq app.
* I've created a sub path in postgres statefulset volume in order to solve the `directory "/var/lib/postgresql/data/pgdata" exists but is not empty` issue
### Suggested Optimizations
* Adding helm values of config and secrets for other environments such as dev, test and stg
* Adding CI configurations to deploy the app to specific env. once it gets merged on specific branches such as circleci and jenkins files.
* Creating Horizontal Pod Autoscaler(HPA) to scale the pods automatically when CPU or Memory reaches a certain threshold
* Creating Vertical Pod Autoscaler(VPA) to analyze and adjust statefulset containers' CPU or Memory requests
* Using RBAC Authorization for users and groups on clusters or namespaces according to their roles
* Adding ssl certs to secure the app
* Making a database seeder job for each environment
* Scheduling a kubernetes cronjob to dump the database into s3
* Setup ELK/EFK stack for better logging
* Setup Prometheus, Grafana and Alert Manager for monitoring
* Using proxy server such as nginx or haproxy
* Setup a VPN to prevent unauthorized access to the k8s cluster
* Calculate the avg num of pod replicas and how many connections pool they use and adjust it
* Create a different database username and password for each app for better potential analysis and debugging.
* Making sure that containers are lightweight and there's no secret data or credentials stored in container and that there's nobody is using latest tag and there's no more than one process is running in a single container and avoid using privileged container and specifying a non-root user for containers to run as.
* Add server side caching for the most requested apis if possible.
* Add connections rate limit into the proxy server with ip tables to prevent/mitigate simple Denial of Service(DOS) attacks.
