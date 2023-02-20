PROJECT DOCUMENTATION
-----------------------------------------------------------------------------------------------
Project: Deploying Wordpress and Mysql with Persistent Volume on Kubernetes
________________________________________________________________________________________________
Steps1: Lauching EC2 instance, installation and configuration of Apache web server and PHP.

1) login into your AWS management console.
   Click on EC2 instances and select launch instances.
   - Enter a unique name to be for your instance for identification
     select an AMI --for this demo i used Ubuntu 20.0
     select t2 small as your instance type.
     Create a key pair for SSH -- i used .ppk for my putty.. you can use .pem for CLI
-
  - For the Networking section select create security group. (inbound rule)
    allow SSH traffic from your instance IP -- to allow only SSH from our current IP
    allow all HTTP access from all IP address -- for them to view your wordpress site with the brower 
Click on launch instance.
---------------------------------------------------------------------------------------------------------------------------
2) SSH into your instance. for CLI clint like ubuntu. 

 navigate to the directory at which your .pem file is located and SSH.
 $ ssh -i <.pem-file> ubuntu@ip address
 successful login ubuntu host.

  - For Putty clint.
    open your putty:
    host name: ubuntu@ip address
    under connection select the + sign at the ssh 
    and under authentication select credentials.. 
    click on browse and select your .ppk file
    then select open.
    Succefully login in @ubuntu host
---------------------------------------------------------------------------------------------------------------------------------------------------
Step2: installion of Docker and k3s on Ec2 instance -- this deployment is running on containers so we need to install docker and manage it on k3s.

1)  For installing Docker run the following commands
    # update the package manager
      $ sudo apt-get update

     # install dependencies
       $ sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common

     # add the Docker GPG key
       $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

     # add the Docker repository
       $ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

     # update the package manager again
       $ sudo apt-get update

     # install Docker
       $ sudo apt-get install -y docker-ce

     # add the current user to the "docker" group
       $ sudo usermod -aG docker $USER  
2) For installation of k3s run the following commands
       $ curl -sfL https://get.k3s.io | sh -
       $ mkdir -p ~/.kube
       $ sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
       $ sudo chown $USER:$USER ~/.kube/config
       $ export KUBECONFIG=~/.kube/config
       $ kubectl get nodes    
------------------------------------------------------------------------------------------------------------------------------

Step3: Create a kustomisation file for our deployment

secretGenerator:
- name: mysql-pass
  literals:
  - password=password
resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml

------------------------------------------------------------------------------------------------------------------------------
Step4: Create a Deployment file for Mysql ---we will be defining the following into a single file;
1) Service
2) PersistentVolumeClaim
3) Deployment




apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim

----------------------------------------------------------------------------------------------------

Step5: Create a Deployment file for Wordpress ---we will be defining the following into a single;
1) Service
2) PersistentVolumeClaim
3) Deployment




apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 80
  selector:
    app: wordpress
    tier: frontend
  type: NodePort
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim

-------------------------------------------------------------------------------------------------------------------------
Step5: Finally you run your kubernetes file.

$ kubectl apply -k . -- to apply kustomization file
 
#check Deployment status
     $ kubectl get deployment
     $ kubectl get service
     $ kubectl get pods
     $ kubectl get all -o wide -- to check all your kubernetes resourses
   
________________________________________________________________________________________________________________________________

*THE END
