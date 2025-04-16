Step-by-Step Guide to Deploy MongoDB and MongoExpress on Amazon EKS using Helm
This guide provides detailed steps to deploy MongoDB and MongoExpress on an Amazon Elastic Kubernetes Service (EKS) cluster using Helm. It assumes you have basic knowledge of Kubernetes, Helm, and AWS. The deployment will include a MongoDB replica set and MongoExpress as a web-based UI client, with an NGINX Ingress Controller for external access. The guide includes your provided output for the Helm repository setup steps.

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
Add the necessary Helm chart repositories for MongoDB, MongoExpress, and the NGINX Ingress Controller.
Run the following commands:
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

Your Output
Here is the output you provided for the Helm repository setup:
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

Initially, helm repo list showed no repositories (Error: no repositories to show), indicating no Helm repositories were configured.
You added the bitnami repository successfully ("bitnami" has been added to your repositories).
You added the ingress-nginx repository successfully ("ingress-nginx" has been added to your repositories).
The helm repo update command fetched the latest chart information from both repositories.
Finally, helm repo list confirmed that both repositories (bitnami and ingress-nginx) were added correctly.


Step 2: Deploy MongoDB using Helm
We’ll use the Bitnami MongoDB Helm chart to deploy a MongoDB replica set for high availability.
Create a Custom Values File for MongoDB
Create a file named mongodb-values.yaml with the following content to customize the MongoDB deployment:
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


architecture: replicaset: Configures MongoDB as a replica set for high availability.
replicaCount: 3: Deploys three MongoDB pods.
persistence: Uses AWS gp3 EBS storage for data persistence.
auth: Enables authentication with a root password and a user for the mydb database.
service: Uses ClusterIP for internal communication.

Install MongoDB
Deploy MongoDB using the Helm chart:
helm install mongodb bitnami/mongodb --values mongodb-values.yaml --namespace mongodb --create-namespace


mongodb: Release name for the Helm deployment.
--namespace mongodb: Deploys in the mongodb namespace, created if it doesn’t exist.

Verify MongoDB Deployment
Check the status of the MongoDB pods:
kubectl get pods -n mongodb

Output should show three MongoDB pods in the Running state, e.g.:
NAME                 READY   STATUS    RESTARTS   AGE
mongodb-0            1/1     Running   0          2m
mongodb-1            1/1     Running   0          2m
mongodb-2            1/1     Running   0          2m

Check the services:
kubectl get svc -n mongodb

Output should include a ClusterIP service for MongoDB, e.g.:
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
mongodb              ClusterIP   10.100.96.208   <none>        27017/TCP   5m


Step 3: Deploy MongoExpress
MongoExpress provides a web-based UI to interact with MongoDB. We’ll deploy it using a Kubernetes manifest, as the Bitnami chart for MongoExpress is less commonly used.
Create a MongoExpress Deployment Manifest
Create a file named mongoexpress.yaml with the following content:
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


ME_CONFIG_MONGODB_SERVER: Points to the MongoDB service DNS (mongodb.mongodb.svc.cluster.local).
ME_CONFIG_MONGODB_ADMINUSERNAME and ME_CONFIG_MONGODB_ADMINPASSWORD: Use the credentials defined in the MongoDB chart.
Service: Exposes MongoExpress internally on port 8081.

Deploy MongoExpress
Apply the manifest:
kubectl apply -f mongoexpress.yaml

Verify MongoExpress Deployment
Check the MongoExpress pod:
kubectl get pods -n mongodb -l app=mongoexpress

Output should show the MongoExpress pod in the Running state, e.g.:
NAME                           READY   STATUS    RESTARTS   AGE
mongoexpress-5d8b7c4d9f-xyz   1/1     Running   0          1m

Check the service:
kubectl get svc -n mongodb -l app=mongoexpress

Output should show the MongoExpress service, e.g.:
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
mongoexpress   ClusterIP   10.100.97.123   <none>        8081/TCP    1m


Step 4: Set Up NGINX Ingress Controller
To access MongoExpress externally, deploy an NGINX Ingress Controller and configure an Ingress resource.
Install NGINX Ingress Controller
Deploy the NGINX Ingress Controller using Helm:
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer


--namespace ingress-nginx: Deploys in the ingress-nginx namespace.
controller.service.type=LoadBalancer: Provisions an AWS Elastic Load Balancer (ELB) for external access.

Verify Ingress Controller
Check the Ingress Controller pods:
kubectl get pods -n ingress-nginx

Output should show the controller pod in the Running state, e.g.:
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7b8c9d8b6f-abc   1/1     Running   0          2m

Get the external address of the LoadBalancer:
kubectl get svc -n ingress-nginx

Output will show the ELB’s external address, e.g.:
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP                              PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.98.456   a12b34c56d789.elb.amazonaws.com   80:31234/TCP,443:31235/TCP   2m

Note the EXTERNAL-IP (e.g., a12b34c56d789.elb.amazonaws.com).

Step 5: Configure Ingress for MongoExpress
Create an Ingress resource to route external traffic to MongoExpress.
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


host: mongoexpress.example.com: Replace with your domain or use the ELB address for testing.
ingressClassName: nginx: Specifies the NGINX Ingress Controller.
service: Routes traffic to the MongoExpress service on port 8081.

Apply the Ingress resource:
kubectl apply -f mongoexpress-ingress.yaml

Verify Ingress
Check the Ingress resource:
kubectl get ingress -n mongodb

Output should show the Ingress, e.g.:
NAME                   CLASS   HOSTS                     ADDRESS                              PORTS   AGE
mongoexpress-ingress   nginx   mongoexpress.example.com   a12b34c56d789.elb.amazonaws.com   80      1m


Step 6: Access MongoExpress

Update DNS: If using a custom domain (e.g., mongoexpress.example.com), update your DNS records to point to the ELB’s EXTERNAL-IP. For testing, you can edit your local /etc/hosts file:
<ELB-EXTERNAL-IP> mongoexpress.example.com


Access MongoExpress: Open a browser and navigate to http://mongoexpress.example.com. You should see the MongoExpress UI.

Log In: Use the credentials admin / adminpassword to log in and interact with the mydb database.


If you encounter a “Not Secure” warning due to a self-signed certificate, ensure your Ingress is configured with a valid TLS certificate (e.g., using AWS Certificate Manager or Let’s Encrypt).

Step 7: Validate and Test MongoDB
To verify MongoDB functionality, connect to a MongoDB pod and perform basic operations.
Access MongoDB Pod
Exec into a MongoDB pod:
kubectl exec -it mongodb-0 -n mongodb -- bash

Run the MongoDB client:
mongo -u admin -p adminpassword

Perform Basic Operations

Switch to Database:
use mydb


Insert Data:
db.mycollection.insert({ "name": "test", "value": 123 })


Query Data:
db.mycollection.find()


Exit:
exit



You can also use MongoExpress to perform these operations via the web UI.

Step 8: Monitor and Scale
Monitor Resources
Use the AWS Management Console or kubectl to monitor resource usage:
kubectl top pods -n mongodb

Check MongoDB logs if needed:
kubectl logs mongodb-0 -n mongodb

Scale MongoDB
To scale the MongoDB replica set, update the replicaCount in mongodb-values.yaml and upgrade the Helm release:
helm upgrade mongodb bitnami/mongodb --values mongodb-values.yaml --namespace mongodb

Scale EKS Cluster
If more resources are needed, scale the EKS worker nodes using the AWS CLI or Console. Scaling EKS Nodes

Step 9: Clean Up (Optional)
To remove the deployment:

Uninstall Helm Releases:
helm uninstall mongodb -n mongodb
helm uninstall ingress-nginx -n ingress-nginx


Delete Namespace:
kubectl delete namespace mongodb ingress-nginx


Delete EKS Cluster (if no longer needed):
aws eks delete-cluster --name <cluster-name> --region <region>




Troubleshooting

Pods Not Running: Check pod logs (kubectl logs <pod-name> -n mongodb) for errors. Ensure the storage class gp3 is available or replace it with a valid one (e.g., gp2).
MongoExpress Connection Issues: Verify the ME_CONFIG_MONGODB_SERVER points to the correct MongoDB service DNS and credentials are correct.
Ingress Not Working: Ensure the NGINX Ingress Controller is running and the ELB is accessible. Check Ingress annotations and DNS settings.
EFS Issues: If using EFS instead of EBS, ensure the EFS CSI driver is installed and configured correctly. EFS CSI Driver


References

Bitnami MongoDB Helm Chart: https://github.com/bitnami/charts/tree/main/bitnami/mongodb
NGINX Ingress Controller: https://kubernetes.github.io/ingress-nginx/
AWS EKS Documentation: https://docs.aws.amazon.com/eks/
Helm Documentation: https://helm.sh/docs/

