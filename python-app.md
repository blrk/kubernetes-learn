### Deploy a python app interacting with postgress db

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
  namespace: default
spec:
  storageClassName: "" 
  volumeName: local-pv-sc
  accessModes:
    - ReadWriteOnce
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
```

