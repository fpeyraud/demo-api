# Demo API with an Infrastructure as a Code

This repository contains an Helm chart creating a service on a Kubernetes VPC with :

- a simple [httpbin API](https://github.com/postmanlabs/httpbin)
- a load balancer
- an horizontal pod autoscaler policy

It is cloud provider agnostic. It has been tested on a home-made micro-k8s single machine "cluster" with [MetalLB](https://metallb.org/) add-on enabled. It should work on other cloud providers such as AWS or GCP or Scaleway Kapsule. Though be careful with the IAMs required to provision LBs and resources with your cloud provider.

## Pre-requisites

In order to test this chart, you must first :

- have access to a Kubernetes VPC
- have a metrics service installed in order to have the HPA work. (Nice tutorial on [eksworkshop](https://www.eksworkshop.com/beginner/080_scaling/deploy_hpa/))
- have `kubectl` command installed and configured to communicate with your VPC
- have `helm` command installed and configured ([tutorial](https://www.eksworkshop.com/beginner/060_helm/helm_intro/install/))

## Ready Set... GOOOOO

```
git clone https://github.com/fpeyraud/demo-api.git
helm install mydemoapi demo-api
```
You can then get the external IP of the service :

```
kubectl get service mydemoapi-demo_api --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}"
```

Then you should be able to browse this IP address in your favorite web browser to check that the service is up and running. You may have to wait for a while before the load balancer gets provisionned since it relies on the cloud provider infrastructure implementation.

## Test autoscaling

In a terminal, launch the HPA monitoring :

```
kubectl get hpa mydemoapi-demo-api -w
```

After a while, in another terminal, create load on the service with a tool like [Siege](https://github.com/JoeDog/siege) :

```
siege http://192.168.0.210
```

Keep Siege running for a minute or two, then stop it. It should be sufficient to let HPA scale up near to the limit imposed in the configuration file. Going back to the HPA monitoring, you should see the process of pod creation and termination which should look like something like this : 

```
NAME                 REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   1%/60%    1         10        1          2m53s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   10%/60%   1         10        1          3m4s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   1%/60%    1         10        1          3m19s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   226%/60%   1         10        1          4m5s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   344%/60%   1         10        4          4m20s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   115%/60%   1         10        6          4m36s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   88%/60%    1         10        6          5m6s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   88%/60%    1         10        8          5m22s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   75%/60%    1         10        9          5m37s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   63%/60%    1         10        9          6m8s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   62%/60%    1         10        9          6m23s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   63%/60%    1         10        9          6m38s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   64%/60%    1         10        9          7m9s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   49%/60%    1         10        9          7m25s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   14%/60%    1         10        9          7m40s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   1%/60%     1         10        9          7m55s
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   1%/60%     1         10        9          12m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   1%/60%     1         10        8          12m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   1%/60%     1         10        3          12m
mydemoapi-demo-api   Deployment/mydemoapi-demo-api   1%/60%     1         10        1          12m
```