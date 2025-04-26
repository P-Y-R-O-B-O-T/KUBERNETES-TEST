# DEVOPS ASSIGNMENT

## Task 1: Check for Existing Taints
* Taints can be checked in 2 ways:
### Method 1 
```bash
kubectl describe nodes
```
* This returns the description of all nodes in the cluster.
```
Name:               worker-1
Roles:              <none>
Labels:             kubernetes.io/hostname=worker-1
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/containerd/containerd.sock
Taints:             <none>
CreationTimestamp:  8h ago
Conditions:
  Type              Status    LastHeartbeatTime    LastTransitionTime    Reason             Message
  Ready             True      8s ago                8h ago                KubeletReady       kubelet is posting ready status

Name:               worker-2
Roles:              <none>
Labels:             kubernetes.io/hostname=worker-2
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/containerd/containerd.sock
Taints:             <none>
CreationTimestamp:  8h ago
Conditions:
  Type              Status    LastHeartbeatTime    LastTransitionTime    Reason             Message
  Ready             True      9s ago                8h ago                KubeletReady       kubelet is posting ready status

Name:               worker-3
Roles:              <none>
Labels:             kubernetes.io/hostname=worker-3
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/containerd/containerd.sock
Taints:             <none>
CreationTimestamp:  8h ago
Conditions:
  Type              Status    LastHeartbeatTime    LastTransitionTime    Reason             Message
  Ready             True      7s ago                8h ago                KubeletReady       kubelet is posting ready status
```
> [!NOTE]
> ### Sections
> * When running the command we get the following info set:
>     - **Name**: Name of the nodes
>     - **Roles**: What roles have been set to nodes which are set at `kubeadm init` or while creation of cluster, but can be set manually as well
>     - **Labels**: Key value tags attached to nodes, these often help in scheduling and maintenance
>     - **Annotations**: Extra key value pairs or tags set and used by administrator to keep a track of for convenience
>     - **Taints**: These are key value pairs that are attached to nodes which help in selective scheduling of pods
>     - **CreationTimestamp**: Self explanatory
>     - **Conditions**: These are used to trace the status of object status, in this case (nodes)
>         - **Type**: Tells the status of the node, it could be anything from ready, not ready, scheduling disabled, unknown
>         - **LastTransitionTime**: This tells when the status of the node changed last time
>         - **Message**: Human friendly error or debug status message to help in tracking down of issues

> [!IMPORTANT]
> ### Observations
> * Cluster was initiated 8h ago
> * All the nodes are healthy and in ready status
> * There are no taints, and no roles for the worker nodes

### Method 2
```bash
kubectl get nodes -o jsonpath="{range .items[*]}{.metadata.name}{':\t'}{.spec.taints}{'\n'}{end}"
```
```
master-node:      [{"key":"node-role.kubernetes.io/control-plane","value":"","effect":"NoSchedule"}]
worker-node-1:    []
worker-node-2:    []
worker-node-3:    []

```
* The command `kubectl get nodes -o jsonpath="..."` retrieves node information from the cluster and formats it using JSONPath queries.
* The query part `{range .items[*]}{.metadata.name}{':\t'}{.spec.taints}{'\n'}{end}` iterates over the nodes, extracting each node's name and taints.

## Task 2 : Apply taint on a node
```bash
kubectl taint nodes worker-1 app=web:NoSchedule
```

> [!NOTE]
> ### Effect of taint
> * The taint stops the scheduling of any pod that on worker-1 if the pod is not tolerant to that taint
> * **NoSchedule** is the value that stops the scheduling of the pods
> * As a matter, no pod without appropriate toleration wont be scheduled on the node

### Verification
```bash
kubectl describe node worker-1
```
```
Name:               worker-1
Roles:              <none>
Labels:             kubernetes.io/hostname=worker-1
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/containerd/containerd.sock
Taints:             app=web:NoSchedule
CreationTimestamp:  8h ago
Conditions:
  Type              Status    LastHeartbeatTime    LastTransitionTime    Reason             Message
  Ready             True      8s ago                8h ago                KubeletReady       kubelet is posting ready status
```

## Task 3 : Nginx Deployment
* First we need to set taints on the rest of the nodes so that pods for nginx does not get scheduled on them
```bash
kubectl taint nodes worker-2 noapp=no-web:NoSchedule
kubectl taint nodes worker-2 noapp=no-web:NoSchedule
```

* Then this manifest creates the deployment for nginx, save it as `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-logging-deployment
  namespace: test
spec:
  selector:
    matchLabels:
      task: nginx-logging
  template:
    metadata:
      labels:
        task: nginx-logging
        namespace: test
    spec:
      tolerations:
        - key: "app"
          value: "web"
          effect: "NoSchedule"
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 20
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/nginx
        - name: log-agent
          image: busybox
          command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/nginx
      volumes:
        - name: log-volume
          emptyDir: {}
```

* Autoscaling manifest, save it as `autoscale.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: autoscale-logging
  namespace: test
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-logging-deployment
  minReplicas: 5
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 5
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 3
      policies:
        - type: Percent
          value: 50
          periodSeconds: 3
    scaleUp:
      stabilizationWindowSeconds: 3
      policies:
        - type: Percent
          value: 80
          periodSeconds: 3
``` 

> [!NOTE]
> ### Observations
> * Here we define a deployment which creates pod which has 2 containers
>     - One container runs nginx as a web service
>     - Other runs simple `tail` command with follow up to the logs files created by nginx
> * Namespace: Placing the deployment and pods in test namespace to create some segregation between other applications running and our current one (assuming as of now but its a real case)
> * Deployment manages the replications using the `matchLabels` parameter and we match all the parameters here with `task` as key 
> * Tolerations:
>     - `worker-2` and `worker-3` have taints as `noapp=no-web:NoSchedule`
>     - `worker-1` has taint as `app=web:NoSchedule`
>     - Deployment pods have tolerations as `app:web:NoSchedule`, which means pods do not have tolerations for `noapp=no-web:NoSchedule` so, they can't be scheduled on node 2 and 3 and in case of node 1, pods have toleration as `app=web:NoSchedule` so node 1 is the only node which will allow pods to get scheduled on. Therefore all the 5 replicas will be scheduled on node 1
> 
> * Volume: It is required for sharing the logs via disk as per the architecture
> * Log sharing:
>     - The nginx container keeps writing the logs to the log file 
>     - The file is present in the defined volume 
>     - The volume is shared by both containers, so busybox container is also able to access logs
>     - Therefore the busybox keeps printing the logs as per the `tail` command with follow argument
> 
> * Readiness probe:
>     - Making http requests to the root server path at port 80 (default nginx and web) starting 5 seconds after the container successfully starts running, with 10s gap between the subsequent requests
> * Liveliness probe:
>     - Making http requests to the root server path at port 80 (default nginx and web) starting 10 seconds after the container successfully starts running, with 10s gap between the subsequent requests
> * Auto scaling:
>     - The pods auto scale up if the avg cpu utilization reaches 80 percent for 3 seconds consecutively
>     - The pods auto scale down if the avg cpu utilization reaches below 50 percent for 3 seconds consecutively
>     - To avoid rapid up scale or down scale which could happen due to minute metrics fluctuation, there is stabilization window of 3 seconds.

* To apply the deployment manifest run:
```bash
kubeclt apply -f deployment.yaml
```

* To apply the deployment manifest run:
```bash
kubeclt apply -f autoscale.yaml
```

* To check if the service is running
```bash
kubectl run client-pod --image=busybox --tty --stdin -- /bin/sh
kubectl exec -it client-pod -- /bin/sh
curl http://nginx-logging-deployment-4f8e1c9b7d-9jqxg.test.pod.cluster.local
curl http://nginx-logging-deployment-4f8e1c9b7d-psl9h.test.pod.cluster.local
curl http://nginx-logging-deployment-4f8e1c9b7d-8r7kw.test.pod.cluster.local
curl http://nginx-logging-deployment-4f8e1c9b7d-x2bfr.test.pod.cluster.local
curl http://nginx-logging-deployment-4f8e1c9b7d-k5d7m.test.pod.cluster.local
```

### service
* Create a manifest for `clusterIP` service, save it as `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-logging-service
  namespace: test
spec:
  selector:
    task: nginx-logging
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

> [!NOTE]
> ### Observations
* The service is of type `clusterIP` which means only applications inside the cluster will be able to reach the service
* This service resides in test namespace serving at port 80 and forwarding to all pods having the label `task: nginx-logging` at port 80
* It uses TCP and not UDP

* Create service
```bash
kubectl apply -f service.yaml
```
