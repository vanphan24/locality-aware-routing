# locality-aware-routing

This repo will walk you through how to to configure locality aware routing on a Consul deployment.

# Pre-reqs

We will use AWS EKS in this example but the same concepts can apply to any cloud managed Kubernetes cluster, like Azure AKS (which may be quicker to deploy).
If using AWS EKS, your AWS VPC will need to have three separate subnets, each in a different Availability Zone. For Azure, an AKS cluster can be created without the need to explcitily create subnets in separate zones.

Create a EKS or AKS Kubernetes cluster. In the cluster creation, select all three subnets (that belong to the different zones)

Note: For AKS, when creating your AKS cluster, make sure you select Zones 1, 2, 3 for the Availability Zones field.

Confirm you Kubernetes cluster nodes reside on different zones.
In the below example, our EKS cluster nodes are running in us-east-2a, us-east-2c, us-east-2c:
kubectl get nodes -L topology.kubernetes.io/zone
NAME                                           STATUS     ROLES      AGE    VERSION                   ZONE   
ip-172-34-123-123-us-east-2.compute.internal   Ready       <none>    5m     v1.25.12-eks-8cccc123     us-east-2a
ip-172-34-123-124-us-east-2.compute.internal   Ready       <none>    5m     v1.25.12-eks-8cccc123     us-east-2b
ip-172-34-123-125-us-east-2.compute.internal   Ready       <none>    5m     v1.25.12-eks-8cccc123     us-east-2c

# Deploy Consul



# Configure three instances of the Counting service to run in three different zones.

In the Deployment of your counting-zone1.yaml file, edit the topology.kubernetes.io/zone field with your first zone, example us-east-2a.
```
      nodeSelector:
        topology.kubernetes.io/zone: <YOUR_FIRST_ZONE>
```


In the Deployment of your counting-zone2.yaml file, edit the topology.kubernetes.io/zone field with your second zone, example us-east-2b.

```
      nodeSelector:
        topology.kubernetes.io/zone: <YOUR_SECOND_ZONE>
```

Lastly, in the Deployment of your counting-zone3.yaml file, edit the topology.kubernetes.io/zone field with your third zone, example us-east-2c.

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

Next, using a proxy-default config entry, we will enable localtiy aware routing, and traffic will only go to the couting service in the same zone as the dashboard service.

```
kubectl apply -f proxy-default-locality.yaml
```

Refresh the browser several times and confirm the counter is from all three zones.
