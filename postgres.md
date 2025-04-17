* postgress pvc
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
* Deployment postgres
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: mtvlabk8su1
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
            claimName: my-pvc
```
* Create service
  ```bash
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: mtvlabk8su1 
spec:
  selector:
    app: postgres 
  ports:
    - protocol: TCP
      port: 5432 
      targetPort: 5432 
  type: ClusterIP 
 ```
