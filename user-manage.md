### Create user and provide access to the cluster
* create key for the user
```bash
mkdir cert && cd cert
```
* create a private key
```bash
openssl genrsa -out ec2-user.key 2048
```
* Create a certificate signing request. username (CN), group(O)
```bash
openssl req -new -key ec2-user.key -out ec2-user.csr -subj "/CN=ec-user/O=cka-preparation"
```
* Sign the CSR with the k8s cluster certificate authority. Usually present in /etc/kubernetes/pki. Need to ahve ca.crt and ca.key
```bash
ls /etc/kubernetes/pki/
apiserver-etcd-client.crt  apiserver-kubelet-client.crt  apiserver.crt  ca.crt  etcd                front-proxy-ca.key      front-proxy-client.key  sa.pub
apiserver-etcd-client.key  apiserver-kubelet-client.key  apiserver.key  ca.key  front-proxy-ca.crt  front-proxy-client.crt  sa.key
```
```bash
openssl x509 -req -in ec2-user.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out ec2-user.crt -days 365 
```
* Create a user in K8s by setting a user entry in kubeconfig for ec2-user. Point tot he crt and key file. finally set a content entry in kubeconfig for ec2-user
```bash
kubectl config set-credentials ec2-user --client-certificate=ec2-user.crt --client-key=ec2-user.key
```
```bash
kubectl config set-context ec2-user-context --cluster=kubernetes --user=ec2-user
```
```bash
kubectl config use-context ec2-user-context
```
```bash
kubectl config current-context
```


