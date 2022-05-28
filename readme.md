# Demo API with an Infrastructure as a Code

This repository contains an Helm chart creating a service on a Kubernetes VPC with :

- a VPC
- a simple [httpbin API](https://github.com/postmanlabs/httpbin)
- a load balancer
- an horizontal pod autoscaler policy

It is based on AWS for the VPC part. The service and autoscaler is provider agnostic. It has been tested on a home-made micro-k8s single machine "cluster" with [MetalLB](https://metallb.org/) add-on enabled. It has also been tested on AWS EKS. It should work on other cloud providers such as Azure AKS, GCP GKE or Scaleway Kapsule with no change concerning the service part. The VPC part must be adapted according to the provider. There are terraform modules for each of these providers.

## Pre-requisites

In order to test this chart, you must first :

- have `terraform` installed ([link](https://learn.hashicorp.com/tutorials/terraform/install-cli))
- have an AWS account with a administrative IAM allowed to create resources,
- have [`aws` CLI](https://aws.amazon.com/fr/cli/) installed and configured with this IAM
- have `kubectl` command installed ([link](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/))
- have `helm` command installed and configured ([tutorial](https://www.eksworkshop.com/beginner/060_helm/helm_intro/install/))
- Clone this repository : `git clone https://github.com/fpeyraud/demo-api.git` and `cd` into it

## provision the VPC

Terraform will create all the resources on AWS to setup the k8s cluster. You can change the number and type of VM used for the worker nodes in the file `eks-cluster.tf`
```
cd vpc
terraform init
terraform apply
```
Answer `yes` to the resource creation confirmation and watch the magic happen... (~10 minutes). At the end of the process, terraform outputs many information concerning the resources it has ust instanciated.

## Configure kubectl 

In order to interact with your new cluster, you must configure `kubectl` based on the information contained in both terraform outputs and AWS console, retreived with the CLI:
```
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
 ```
## Deploy the metrics server

The metric server is necessary in order to make Horizontal Pod Autoscaler work properly, since it is based on CPU and memory usage.
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```
## Deploy the kubernetes dashboard
As a sort of _Hello World_ for Kuberneted, let's deploy the k8s dashboard and see if everything is OK
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

## Test the dashboard
The dashboard is not public on the internet. It is necessary to use a (kub-)proxy to access it :
```
kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &
aws eks get-token --cluster-name vpc-demo-jYZQfWKl | jq -r '.status.token'
```
This last command gives you the connection token for the kubernetes dashboard. Browse to the URL http://localhost:8080/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

On the login page, select "token" option and paste the token you've just extracted right above.

## Let's go for a real and scalable service

We use helm in order to deploy a simple api on our cluster, this time with a public internet access :
```
cd ..
helm install mydemoapi demo-api
```
You can then get the external IP of the service :

```
kubectl get service mydemoapi-demo_api
```
```
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP
mydemoapi-demo-api   LoadBalancer   172.20.55.15   ac...71.eu-west-3.elb.amazonaws.com   80:32273/TCP   4m22s
```

Then you should be able to browse this IP address/DNS entry in your favorite web browser to check that the service is up and running. You may have to wait for a while before the load balancer gets provisionned since it relies on the cloud provider infrastructure implementation.

## Test autoscaling

In a terminal, launch the HPA monitoring :

```
kubectl get hpa mydemoapi-demo-api -w
```

After a while, in another terminal, create load on the service with a tool like [Siege](https://github.com/JoeDog/siege) :

```
siege http://192.168.0.210
```

Keep Siege running for a minute or two, then stop it. It should be sufficient to let HPA scale up near to the limit imposed in the configuration file (well it has been sufficient on my tests on my own microk8s, but not on AWS VPC). Going back to the HPA monitoring, you should see the process of pod creation and termination which should look like something like this : 

```
NAME                 REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   5%/60%    1         10        1          5m13s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   9%/60%    1         10        1          5m51s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   6%/60%    1         10        1          6m52s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   7%/60%    1         10        1          7m53s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   5%/60%    1         10        1          8m54s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   75%/60%   1         10        1          9m56s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   75%/60%   1         10        2          10m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   59%/60%   1         10        2          10m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   59%/60%   1         10        2          11m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   0%/60%    1         10        2          12m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   0%/60%    1         10        2          17m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   0%/60%    1         10        1          17m
```

If siege cannot trigger autoscaling from your workstation, you can try to load the service from within the VPC :
```
kubectl run -i --tty load-generator --image=busybox /bin/sh
while true; do wget -O - http://mydemoapi-demo-api/ >>/dev/null; done
```
This is the way the HPA logs above has been obtained. It's pretty hard to heavily load the service this way. You could also install `siege` or `ab` in the alpine image

## Cleanup

```
helm uninstall mydemoapi
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/r`eleases/download/v0.3.6/components.yaml
```
answer `yes` to the destroy confirmation and look at the world collapsing just before your eyes.

## Conclusion

This workshop demonstrates that it's pretty easy to setup a full kubernetes VPC on a cloud provider infrastructure. It's a matter of fact that Infrastructure as Code saves _a lot_ of manipulations. Could we make it even shorter ? Probably. A solution involving [`eksctl`](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) can describe a basic k8s VPC in a dozen of lines.

It has to be understood that such a demo is not ready for production ! It lacks a lot of things such as monitoring, logs collection and so on. Nevertheless it's a good basement you could build up on.