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


