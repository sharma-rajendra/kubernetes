follow below link for docker hub authentication
https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/


------
Create a Secret based on existing Docker credentials for pulling image from your docker repository

docker login 

cat ~/.docker/config.json
The output contains a section similar to this:


{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "c3R...zE2"
        }
    }
}


sudo kubectl create secret generic regcred --from-file=.dockerconfigjson=config.json --type=kubernetes.io/dockerconfigjson

kubectl get secret regcred --output=yaml
The value of the .dockerconfigjson field is a base64 representation of your Docker credentials.

echo "" | base64 --decode

outpu look like below:
rajsharma:PASSXXXXWORD




Here is a configuration file for a Pod that needs access to your Docker credentials in regcred:

Create private-reg-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: rajsharma/nginx-kube:v1
  imagePullSecrets:
  - name: regcred



Download the above file:
wget -O my-private-reg-pod.yaml https://k8s.io/examples/pods/private-reg-pod.yaml



kubectl apply -f my-private-reg-pod.yaml
kubectl get pod private-reg



############################################################################################################

					Create custome nginx image


############################################################################################################

Create custome nginx image

vim Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html


cat index.html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Hello World - Nginx Docker</title>
    <style>
        h1{
            font-weight:lighter;
            font-family: Arial, Helvetica, sans-serif;
        }
    </style>
</head>
<body>
    
    <h1>
        Hello World
    </h1>

</body>
</html>


Build image from Dockerfile and push into docker hub
sudo docker build -t nginx-kube . (nginx-kube is name of image)

sudo docker run -d nginx-kube

sudo docker ps 

sudo docker commit nginx-kube-container-id rajsharma/nginx-kube:v1

sudo docker push  rajsharma/nginx-kube:v1



############################################################################################################

					Create kubernetes Service and Deployment

############################################################################################################

vim nginx-deployment-service.yml


apiVersion: v1
kind: Service
metadata:
 name: nginx-service [Service name]
 labels:
   app: nginx
spec:
 type: NodePort
 ports:
 - port: 80 [kubernetes cluster port]
   nodePort: 30004 [kubernetes pod port]
   protocol: TCP
 selector:
   app: nginx
########################

vim nginx-deployment.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment [Nginx deployment name]
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: rajsharma/nginx-kube:v1 [Image pull from your dockerhub]
        imagePullPolicy: Always [It should be always because your code reflect on prod with this]
        ports:
        - containerPort: 80
      imagePullSecrets: [We have created new pod for credentials before]
      - name: regcred [credentials pod name]


Now create service and deployment 

kubectl create -f nginx-deployment-service.yml;kubectl create -f nginx-deployment.yml




############################################################################################################

					Rollout new code on kubernetes cluster

############################################################################################################


raj@k8s-master:/techcode/nginx-docker$ sudo docker run -d rajsharma/nginx-kube:v1



sudo docker exec -it cocky_wu /bin/sh

cd /usr/share/nginx/html/

vi index.html (add some paramenters as you need)




sudo docker commit cocky_wu rajsharma/nginx-kube:v2


sudo docker push  rajsharma/nginx-kube:v2


kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY   STATUS        RESTARTS   AGE
default       nginx-deployment-8f8cb4f66-b4hsc        1/1     Running       0          3s
default       nginx-deployment-8f8cb4f66-vmwjn        1/1     Running       0          3s



kubectl set image deployment  nginx-deployment nginx=rajsharma/nginx-kube:v2 --record
deployment.extensions/nginx-deployment image updated

kubectl get pods --all-namespaces
NAMESPACE     NAME                                    READY   STATUS              RESTARTS   AGE
default       nginx-deployment-89b4c87ff-rlnlq        0/1     ContainerCreating   0          3s
default       nginx-deployment-8f8cb4f66-b4hsc        1/1     Running             0          39s
default       nginx-deployment-8f8cb4f66-vmwjn        1/1     Running             0          39s



