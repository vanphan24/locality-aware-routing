# locality-aware-routing

This repo will walk you through how to to configure locality aware routing on a Consul deployment.

# Pre-reqs

Locality Aware Routing is an enterprise feature and will require an Enterprise license.

We will use AWS EKS in this example but the same concepts can apply to any cloud managed Kubernetes cluster, like Azure AKS (which may be quicker to deploy).
If using AWS EKS, your AWS VPC will need to have three separate subnets, each in a different Availability Zone. For Azure, an AKS cluster can be created without the need to explcitily create subnets in separate zones.

Create a EKS or AKS Kubernetes cluster. In the cluster creation...
- For EKS, select all three subnets (that belong to the different zones).
        
- For AKS, when creating your AKS cluster, make sure you select Zones 1, 2, 3 for the Availability Zones field.

Confirm you Kubernetes cluster nodes reside on different zones.
In the below example, our EKS cluster nodes are running in us-east-2a, us-east-2c, us-east-2c:
```
kubectl get nodes -L topology.kubernetes.io/zone
NAME                                           STATUS     ROLES      AGE    VERSION                   ZONE   
ip-172-31-2-109.us-east-2.compute.internal    Ready       <none>    5m     v1.25.12-eks-8cccc123     us-east-2a
ip-172-31-29-111.us-east-2.compute.internal   Ready       <none>    5m     v1.25.12-eks-8cccc123     us-east-2b
ip-172-31-46-215.us-east-2.compute.internal   Ready       <none>    5m     v1.25.12-eks-8cccc123     us-east-2c
```
# Deploy Consul

Clone repo
```git clone https://github.com/vanphan24/locality-aware-routing.git```

Navigate to clones directory
```cd locality-aware-routing```

Set Consul license 
```
kubectl create secret generic license --from-literal=key=<ADD_YOUR_LICENSE_HERE>
```
Deploy Consul
```
helm install consul hashicorp/consul --version 1.3.0-rc1 --values consul-value.yaml 
```
Confirm Consul install
```
kubectl get pod
NAME                                                  READY   STATUS    RESTARTS   AGE
consul-consul-connect-injector-5db9db87d9-rlxhn       1/1     Running   0          5m
consul-consul-mesh-gateway-78b89c7b6-nj22p            1/1     Running   0          5m
consul-consul-server-0                                1/1     Running   0          5m
consul-consul-webhook-cert-manager-58db5d5ddd-6wmgm   1/1     Running   0          5m
```





# Configure three instances of the Counting service to run in three different zones.

In the Deployment of your counting-zone1.yaml file, edit the ```topology.kubernetes.io/zone``` field with your first zone, example for AWS zone ```us-east-2a```.
```
      nodeSelector:
        topology.kubernetes.io/zone: <YOUR_FIRST_ZONE>
```


In the Deployment of your counting-zone2.yaml file, edit the ```topology.kubernetes.io/zone``` field with your second zone, example for AWS zone ```us-east-2b```.

```
      nodeSelector:
        topology.kubernetes.io/zone: <YOUR_SECOND_ZONE>
```

Lastly, in the Deployment of your counting-zone3.yaml file, edit the ```topology.kubernetes.io/zone``` field with your third zone, example for AWS zone ```us-east-2c```.

```
      nodeSelector:
        topology.kubernetes.io/zone: <YOUR_THIRD_ZONE>
```

# Deploy both the dashboard and counting service in the first availabiltiy zone.
```
kubectl apply -f dashboard.yaml -f counting-zone1.yaml 
```

Run port-forward on dashboard service to confirm dashboard can reach counting
```
kubectl port-forward service/dashboard 9002:9002 
```

Open browser and confirm counter is incrementing on the Dashbaord web UI.


Deploy the second counting instance.
```
kubectl apply -f counting-zone2.yaml -f counting-zone2.yaml 
```

Refresh the browser and confirm the counter on the Dashboard is not incrementing sequencially. This would confirm the dashboard service is reflecting the counter from different instances in different zones.

Deploy the third counting instance.
```
kubectl apply -f counting-zone3.yaml -f counting-zone3.yaml 
```

Refresh the browser several times and confirm the counter is from all three zones.
You can also confirm the zone by the name of the counter pod beneath the number count.

Confirm all three counting instances are running on different nodes:
```
kubectl get pod -o wide
NAME                                                  READY   STATUS     RESTARTS   AGE     IP              NODE 
counting-zone1-f5b764fb6-4cjrl                        2/2     Running    0          2m10s   172.31.0.64     ip-172-31-2-109.us-east-2.compute.internal            
counting-zone2-b9897fb5c-cqjkr                        2/2     Running    0          2m45s   172.31.30.109   ip-172-31-29-111.us-east-2.compute.internal   
counting-zone3-cf8765cd4-lvvq8                        2/2     Running    0          3m20s   172.31.38.124   ip-172-31-46-215.us-east-2.compute.internal 
dashboard-64f7f8cb5-kp498                             2/2     Running    0          2m10s   172.31.25.179   ip-172-31-29-111.us-east-2.compute.internal
```

Next, using a proxy-default config entry, we will enable localtiy aware routing, and traffic will only go to the couting service in the same zone as the dashboard service.

```
kubectl apply -f proxy-default-locality.yaml
```

Refresh the browser several times and confirm the counter is from all three zones.
