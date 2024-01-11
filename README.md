# Vertical Pod Autoscaler (VPA)

- VPA automatically adjust the CPU and Memory reservations for our pods to help ```right size``` our applications.
- This adjustment can improve cluster resource utilizations and free up CPU and Memory for other pods.

### Benefits
- Cluster nodes are used efficiently, because pods used exactly what they need.
- Pods are sheduled onto nodes that have appropriate resources available.
- We don't have to run time-consuming benchmarking tasks to determine the correct values for CPU and Memory request.
- Maintenance time is reduced, because the autoscaler can adjust **CPU and Memory** request over time without any actions on your part.

### VPA Components

#### VPA Admission Hook 
Every pod submitted to the k8s cluster goes through this webhook automatically. Which checks whether a VerticalPodAutoscaler object is referencing this pod or one of its parents(A ReplicaSets, Deployment etc)

#### VPA Recommender
Connects to the metric-server in the cluster, fetches historical and current usages data(CPU and Memory) for each VPA enabled pod and generates recommendation for scaling up or down the request and limit of these pods.

#### VPA Updater
Runs every 1 minutes, If a pod is not running in the calculated recommendation range, its evict the current running version of this pods. So it can restart and go through the VPA admission webhook which will change the CPU and Memory settings for it, before it can start.

**metric-server should be installed**

#### Installed VPA Components

1. Clone the Repo

```
git clone https://github.com/kubernetes/autoscaler.git
```

2. Navigate 
```
cd autoscaler/vertical-pod-autoscaler/
```

3. To Install 
```
./hack/vpa-up.sh

customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalercheckpoints.autoscaling.k8s.io created
customresourcedefinition.apiextensions.k8s.io/verticalpodautoscalers.autoscaling.k8s.io created
clusterrole.rbac.authorization.k8s.io/system:metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:vpa-actor created
clusterrole.rbac.authorization.k8s.io/system:vpa-status-actor created
clusterrole.rbac.authorization.k8s.io/system:vpa-checkpoint-actor created
clusterrole.rbac.authorization.k8s.io/system:evictioner created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-actor created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-status-actor created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-checkpoint-actor created
clusterrole.rbac.authorization.k8s.io/system:vpa-target-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-target-reader-binding created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-evictioner-binding created
serviceaccount/vpa-admission-controller created
serviceaccount/vpa-recommender created
serviceaccount/vpa-updater created
clusterrole.rbac.authorization.k8s.io/system:vpa-admission-controller created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-admission-controller created
clusterrole.rbac.authorization.k8s.io/system:vpa-status-reader created
clusterrolebinding.rbac.authorization.k8s.io/system:vpa-status-reader-binding created
deployment.apps/vpa-updater created
deployment.apps/vpa-recommender created
Generating certs for the VPA Admission Controller in /tmp/vpa-certs.
Certificate request self-signature ok
subject=CN = vpa-webhook.kube-system.svc
Uploading certs to the cluster.
secret/vpa-tls-certs created
Deleting /tmp/vpa-certs.
deployment.apps/vpa-admission-controller created
service/vpa-webhook created
```

4. To Verify
```
kubectl get pods -n kube-system

k get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS      AGE
vpa-admission-controller-8f769c59f-5nxrj   1/1     Running   0             100s
vpa-recommender-589c854d7b-l7bw2           1/1     Running   0             102s
vpa-updater-5786bdb8c-kwnjr                1/1     Running   0             102s
```

5. To Uninstall
```
./hack/vpa-down.sh
```

> [!Note]
> In deployment we only define request and in VPA we define limits.
> It performs in RollingUpdate Strategy, its doesn't down entire applications.
> If we have only one pod, unless we manually delete that pod, it will not launch new pod with VPA recommended CPU and Memory consideration the application availibility senarios.

#### Test Sample Applications

```
k apply -f manifests/deployment.yaml 
deployment.apps/hamster created

k get pods
NAME                       READY   STATUS    RESTARTS   AGE
hamster-59cc68d575-d5l62   1/1     Running   0          28s
hamster-59cc68d575-t77lf   1/1     Running   0          28s
```

```
k apply -f manifests/vpa.yaml 
verticalpodautoscaler.autoscaling.k8s.io/hamster-vpa created

k get vpa
NAME          MODE   CPU    MEM       PROVIDED   AGE
hamster-vpa   Auto   511m   262144k   True       18s
```

```
kubectl get pods --watch
NAME                       READY   STATUS    RESTARTS   AGE
hamster-59cc68d575-blb4h   1/1     Running   0          54s
hamster-59cc68d575-t77lf   1/1     Running   0          2m27s
hamster-59cc68d575-t77lf   1/1     Terminating   0          2m33s
hamster-59cc68d575-cvkh7   0/1     Pending       0          1s
hamster-59cc68d575-cvkh7   0/1     Pending       0          1s
hamster-59cc68d575-t77lf   0/1     Terminating   0          3m5s
hamster-59cc68d575-t77lf   0/1     Terminating   0          3m5s
hamster-59cc68d575-t77lf   0/1     Terminating   0          3m5s
```

```
k get vpa --watch
NAME          MODE   CPU    MEM       PROVIDED   AGE
hamster-vpa   Auto   548m   262144k   True       2m52s
hamster-vpa   Auto   587m   262144k   True       3m8s
```

#### To Uninstall VPA Components (Optional)

```
./hack/vpa-down.sh 

```