Step-by-Step Guide to Deploy MongoDB and MongoExpress on Amazon EKS using Helm
This guide provides detailed steps to deploy MongoDB and MongoExpress on an Amazon Elastic Kubernetes Service (EKS) cluster using Helm. It assumes you have basic knowledge of Kubernetes, Helm, and AWS. The deployment includes a MongoDB replica set and MongoExpress as a web-based UI client, with an NGINX Ingress Controller for external access.
Prerequisites
Before starting, ensure you have the following:

AWS CLI: Installed and configured with access to your AWS account. AWS CLI Installation Guide

kubectl: Installed and configured to interact with your EKS cluster. kubectl Installation Guide

Helm: Installed on your local machine. Helm Installation Guide

An EKS Cluster: A running EKS cluster. If you don’t have one, create it using the AWS CLI or AWS Management Console. EKS Getting Started Guide

IAM Permissions: Ensure your AWS user or role has permissions to manage EKS, EC2, and related resources.

kubectl Configured: Update your kubeconfig to connect to the EKS cluster:
aws eks update-kubeconfig --region <region> --name <cluster-name>



Verify your setup:
kubectl get nodes
helm version

Step 1: Set Up Helm Repositories
Add the necessary Helm chart repositories for MongoDB, MongoExpress, and the NGINX Ingress Controller:
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

Provided Output
rahulranjan@Rahuls-MacBook-Air ~ % helm repo list      
Error: no repositories to show
rahulranjan@Rahuls-MacBook-Air ~ % helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
rahulranjan@Rahuls-MacBook-Air ~ % helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
rahulranjan@Rahuls-MacBook-Air ~ % helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
rahulranjan@Rahuls-MacBook-Air ~ % helm repo list                                                        
NAME         	URL                                       
bitnami      	https://charts.bitnami.com/bitnami        
ingress-nginx	https://kubernetes.github.io/ingress-nginx

Explanation of Output

Initially, helm repo list showed no repositories (Error: no repositories to show).
The bitnami and ingress-nginx repositories were added successfully.
helm repo update fetched the latest chart information.
The final helm repo list confirmed both repositories were added correctly.

Step 2: Deploy MongoDB using Helm
Use the Bitnami MongoDB Helm chart to deploy a MongoDB replica set.
Create a Custom Values File
Create a file named mongodb-values.yaml:
architecture: replicaset
replicaCount: 3
persistence:
  enabled: true
  storageClass: "gp3"
  size: 8Gi
auth:
  enabled: true
  rootPassword: "helloworld"
  usernames:
    - "admin"
  passwords:
    - "adminpassword"
  databases:
    - "mydb"
service:
  type: ClusterIP

Install MongoDB
Deploy MongoDB:
helm install mongodb bitnami/mongodb --values mongodb-values.yaml --namespace mongodb --create-namespace

Verify MongoDB Deployment
Check MongoDB pods:
kubectl get pods -n mongodb

Expected output:
NAME                 READY   STATUS    RESTARTS   AGE
mongodb-0            1/1     Running   0          2m
mongodb-1            1/1     Running   0          2m
mongodb-2            1/1     Running   0          2m

Check services:
kubectl get svc -n mongodb

Expected output:
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
mongodb              ClusterIP   10.100.96.208   <none>        27017/TCP   5m

Step 3: Deploy MongoExpress
Deploy MongoExpress using a Kubernetes manifest for a web-based UI.
Create MongoExpress Manifest
Create a file named mongoexpress.yaml:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongoexpress
  namespace: mongodb
  labels:
    app: mongoexpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongoexpress
  template:
    metadata:
      labels:
        app: mongoexpress
    spec:
      containers:
      - name: mongoexpress
        image: mongo-express:latest
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_SERVER
          value: "mongodb.mongodb.svc.cluster.local"
        - name: ME_CONFIG_MONGODB_PORT
          value: "27017"
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          value: "admin"
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          value: "adminpassword"
        - name: ME_CONFIG_MONGODB_ENABLE_ADMIN
          value: "true"
---
apiVersion: v1
kind: Service
metadata:
  name: mongoexpress
  namespace: mongodb
spec:
  selector:
    app: mongoexpress
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
  type: ClusterIP

Deploy MongoExpress
Apply the manifest:
kubectl apply -f mongoexpress.yaml

Verify MongoExpress Deployment
Check the pod:
kubectl get pods -n mongodb -l app=mongoexpress

Expected output:
NAME                           READY   STATUS    RESTARTS   AGE
mongoexpress-5d8b7c4d9f-xyz   1/1     Running   0          1m

Check the service:
kubectl get svc -n mongodb -l app=mongoexpress

Expected output:
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
mongoexpress   ClusterIP   10.100.97.123   <none>        8081/TCP    1m

Step 4: Set Up NGINX Ingress Controller
Deploy an NGINX Ingress Controller for external access.
Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer

Verify Ingress Controller
Check pods:
kubectl get pods -n ingress-nginx

Expected output:
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7b8c9d8b6f-abc   1/1     Running   0          2m

Get the external address:
kubectl get svc -n ingress-nginx

Expected output:
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP                              PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.98.456   a12b34c56d789.elb.amazonaws.com   80:31234/TCP,443:31235/TCP   2m

Note the EXTERNAL-IP (e.g., a12b34c56d789.elb.amazonaws.com).
Step 5: Configure Ingress for MongoExpress
Create an Ingress resource to route external traffic.
Create Ingress Manifest
Create a file named mongoexpress-ingress.yaml:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mongoexpress-ingress
  namespace: mongodb
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: "mongoexpress.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mongoexpress
            port:
              number: 8081

Apply Ingress
kubectl apply -f mongoexpress-ingress.yaml

Verify Ingress
kubectl get ingress -n mongodb

Expected output:
NAME                   CLASS   HOSTS                     ADDRESS                              PORTS   AGE
mongoexpress-ingress   nginx   mongoexpress.example.com   a12b34c56d789.elb.amazonaws.com   80      1m

Step 6: Access MongoExpress

Update DNS: If using a custom domain (e.g., mongoexpress.example.com), update DNS records to point to the ELB’s EXTERNAL-IP. For testing, edit /etc/hosts:
<ELB-EXTERNAL-IP> mongoexpress.example.com


Access MongoExpress: Open http://mongoexpress.example.com in a browser to see the MongoExpress UI.

Log In: Use credentials admin / adminpassword to interact with the mydb database.



Note: If you see a “Not Secure” warning, configure a valid TLS certificate (e.g., using AWS Certificate Manager or Let’s Encrypt).

Step 7: Validate and Test MongoDB
Connect to a MongoDB pod to verify functionality.
Access MongoDB Pod
kubectl exec -it mongodb-0 -n mongodb -- bash
mongo -u admin -p adminpassword

Perform Basic Operations
use mydb
db.mycollection.insert({ "name": "test", "value": 123 })
db.mycollection.find()
exit

You can also use MongoExpress for these operations via the web UI.
Step 8: Monitor and Scale
Monitor Resources
kubectl top pods -n mongodb
kubectl logs mongodb-0 -n mongodb

Scale MongoDB
Update replicaCount in mongodb-values.yaml and run:
helm upgrade mongodb bitnami/mongodb --values mongodb-values.yaml --namespace mongodb

Scale EKS Cluster
Scale worker nodes via AWS CLI or Console. Scaling EKS Nodes
Step 9: Clean Up (Optional)
Uninstall Helm releases:
helm uninstall mongodb -n mongodb
helm uninstall ingress-nginx -n ingress-nginx

Delete namespaces:
kubectl delete namespace mongodb ingress-nginx

Delete EKS cluster (if needed):
aws eks delete-cluster --name <cluster-name> --region <region>

Troubleshooting

Pods Not Running: Check logs (kubectl logs <pod-name> -n mongodb). Ensure gp3 storage class is available or use gp2.
MongoExpress Connection Issues: Verify ME_CONFIG_MONGODB_SERVER and credentials.
Ingress Not Working: Ensure NGINX Ingress Controller is running and ELB is accessible. Check annotations and DNS.
EFS Issues: If using EFS, ensure the EFS CSI driver is installed. EFS CSI Driver

References

Bitnami MongoDB Helm Chart
NGINX Ingress Controller
AWS EKS Documentation
Helm Documentation

