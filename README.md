# Implementing CI/CD for Multicluster Kubernetes with GitOps in a Hub and Spoke Architecture

## Hub And Spoke Architecture
Hub and spoke architecture is like having a main control center (hub) that takes care of several connected points (spokes). In the world of Kubernetes clusters, you pick one cluster to be the main hub, and it's in charge of handling and organizing other clusters, which are the spokes. The hub does the management work, and the spokes follow its lead, creating a coordinated system.

## Need for ArgoCD(Gitops)
* ArgoCD helps maintain desired state in Kubernetes clusters. In regular CI/CD pipelines, after code is deployed there is no ongoing monitoring to ensure the cluster state matches what was originally deployed. If someone manually changes something small in the cluster, it can be hard to know who made the change or revert it.

* ArgoCD continuously monitors the cluster and compares its state to the desired state defined in git. If any drift is detected where the live cluster no longer matches the git configuration, ArgoCD will automatically revert the cluster to match the desired state in git. This allows ArgoCD to maintain intended state and gives visibility if any unwanted changes occur, since the change will show up as a difference against git that ArgoCD corrects.

* By continuously keeping the cluster aligned with git config, ArgoCD both maintains desired state and provides an audit log if any manual changes are detected.

## EKS Cluster Creation

Now execute the below commands in your local terminal after configuring AWS and installing eksctl.

```
eksctl create cluster --name hub-cluster --region us-east-1
eksctl create cluster --name spoke-cluster-1 --region us-east-1
eksctl create cluster --name spoke-cluster-2 --region us-east-1

```

## Update the kube config

```
aws eks update-kubeconfig --name hub-cluster --region us-east-1
aws eks update-kubeconfig --name spoke-cluster-1 --region us-east-1 
aws eks update-kubeconfig --name spoke-cluster-2 --region us-east-1 
```
Now this allows us to communicate to the API servers of theh repective clusters using kubectl

## switch to our hub-cluster
```
Kubectl config get-contexts
kubectl config use-context "hub-cluster context"
kubectl config current-context 

```

## Install ArgoCD in hubCluster

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

```

When we installed argocd so the deployment uses all the configuration as environment variables from config maps and we want to run Argocd in http mode instead of https to avoid using a ssl certificate in my case since I will delete this once this is done.

execute the below steps

```
kubectl get cm -n argocd
kubectl edit configmap argocd-cmd-params-cm -n argocd
```
And in that above configmap add 
```
data:
 server.insecure: "false"
 
```

Once it is done, now we can run our ArgoCD in insecure mode.

## Edit our argocd svc from cluster IP

```
kubectl edit svc argocd-server -n argocd
```
Edit our svc from clusterIP which is created by default, to type NodePort to access our argocd using hubcluster node.By default, services in Kubernetes are only accessible inside the cluster using ClusterIP. NodePort makes the service accessible externally using the IP address of any node (worker machine) in the cluster.Our EKS cluster is running on EC2 instances rather than Fargate servers. That means each node has a public IP address we can use to access NodePort services.Even if the Argocd pods are not running on the specific node we target, Kubernetes will route the traffic internally to the correct pods. This works because of kube-proxy, which handles networking and traffic routing between pods across different nodes.So in simple terms - we can take the private ClusterIP service and expose it externally using NodePort. Then we can access the Argocd server using any node's public IP address and the port number that gets assigned. Kubernetes will handle routing the request to the correct pod no matter which node we target.


## Adding Clusters to ArgoCD

Once after the installation when you access the argocd UI using NodePort svc we can see the hub cluster in it. To add other clusters so that it can monitor those we need to first Install ArgoCD CLI 

--> Follow the instructions in this link to install [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/#windows).

```
argocd login 54.227.189.136:30812
argocd cluster add <spoke-cluster-1-context> --server 54.227.189.136:30812
argocd cluster add <spoke-cluster-2-context> --server 54.227.189.136:30812

```

