### Deploy a python app interacting with postgress db
#### Storage class
```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-pv-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
parameters:
  fsType: ext3
volumeBindingMode: WaitForFirstConsumer
```
#### Create a pv
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-pv-sc 
  local:
    path: /local-pv 
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - kubeadm-node1 
```
#### Create a Secret for the PostgreSQL Password
```yml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: default
type: Opaque
data:
  POSTGRES_PASSWORD: bXlwYXNzd29yZA==
```
#### Create a Persistent Volume
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-pv-sc
  resources:
    requests:
      storage: 1Gi
```
#### Create a PostgreSQL Deployment
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
        - name: init-data-dir
          image: busybox
          command:
            - /bin/sh
            - -c
            - rm -rf /var/lib/postgresql/data/* || true; mkdir -p /var/lib/postgresql/data
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: "mydatabase"
            - name: POSTGRES_USER
              value: "myuser"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - mountPath: "/var/lib/postgresql/data"
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: volume
                    operator: In
                    values:
                      - local-pv
```
#### Test the DB creation
```bash
kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
postgres-78578b8777-nlmnv   1/1     Running   0          4m41s
[root@kubeadm ~]# kubectl describe pod postgres-78578b8777-nlmnv
Name:             postgres-78578b8777-nlmnv
Namespace:        default
Priority:         0
Service Account:  default
Node:             kubeadm-node1/192.168.122.140
Start Time:       Wed, 22 Jan 2025 16:49:14 +0000
Labels:           app=postgres
                  pod-template-hash=78578b8777
Annotations:      cni.projectcalico.org/containerID: a0b1d4a31dc472cd8691cfcae210f90fd2c292323da4cb90d69af7517afe8439
                  cni.projectcalico.org/podIP: 10.244.98.215/32
                  cni.projectcalico.org/podIPs: 10.244.98.215/32
Status:           Running
IP:               10.244.98.215
IPs:
  IP:           10.244.98.215
Controlled By:  ReplicaSet/postgres-78578b8777
Init Containers:
  init-data-dir:
    Container ID:  containerd://336f19a4a13eb78fc4f9203484fdc9b25c33b842b126f22fcf32180375733aba
    Image:         busybox
    Image ID:      docker.io/library/busybox@sha256:a5d0ce49aa801d475da48f8cb163c354ab95cab073cd3c138bd458fc8257fbf1
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/sh
      -c
      rm -rf /var/lib/postgresql/data/* || true; mkdir -p /var/lib/postgresql/data
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 22 Jan 2025 16:49:18 +0000
      Finished:     Wed, 22 Jan 2025 16:49:18 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/lib/postgresql/data from postgres-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zxdwx (ro)
Containers:
  postgres:
    Container ID:   containerd://73f1eb68d99dd65577200f6237d1581535b3f3dad6ea02b8b865437f8e054446
    Image:          postgres:15
    Image ID:       docker.io/library/postgres@sha256:68bb947ec37e6cfd5486c51ecdd122babc3ddaedb490acb913130a7e325d36c5
    Port:           5432/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 22 Jan 2025 16:49:19 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      POSTGRES_DB:        mydatabase
      POSTGRES_USER:      myuser
      POSTGRES_PASSWORD:  <set to the key 'POSTGRES_PASSWORD' in secret 'postgres-secret'>  Optional: false
    Mounts:
      /var/lib/postgresql/data from postgres-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zxdwx (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  postgres-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  postgres-pvc
    ReadOnly:   false
  kube-api-access-zxdwx:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  5m5s  default-scheduler  Successfully assigned default/postgres-78578b8777-nlmnv to kubeadm-node1
  Normal  Pulling    5m4s  kubelet            Pulling image "busybox"
  Normal  Pulled     5m1s  kubelet            Successfully pulled image "busybox" in 2.497s (2.497s including waiting). Image size: 2167089 bytes.
  Normal  Created    5m1s  kubelet            Created container init-data-dir
  Normal  Started    5m1s  kubelet            Started container init-data-dir
  Normal  Pulled     5m1s  kubelet            Container image "postgres:15" already present on machine
  Normal  Created    5m    kubelet            Created container postgres
  Normal  Started    5m    kubelet            Started container postgres
```


