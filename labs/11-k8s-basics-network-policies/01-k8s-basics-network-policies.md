# Pod Networking Lab

- [Pod Networking Lab](#pod-networking-lab)
  - [Objectives](#objectives)
  - [Deploy Pods](#deploy-pods)
  - [Check Pods default communications](#check-pods-default-communications)
  - [Define and Apply Network Policy](#define-and-apply-network-policy)
  - [Check Network Policy](#check-network-policy)

---

## Objectives

- Learn about Kubernetes Network Policy.
- Define and apply Network Policy.
- Check Network Policy behaviour.

---
...

The figure below highlights the Kubernetes Network Policy concept:  

![Network Policy](pics/network-policy.png)

---

kubectl apply -f 01-sql-pv.yaml
  persistentvolume/mysql-pv-volume created
  persistentvolumeclaim/mysql-pv-claim created
kubectl apply -f 01-sql-deployment.yaml
  service/mysql created
  deployment.apps/mysql created
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -ppassword
  If you don't see a command prompt, try pressing enter.
  mysql> 

  CREATE DATABASE labs_db;
  CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
  CREATE DATABASE labs_db CHARACTER SET utf8 COLLATE utf8_general_ci;

  USE labs_db;

  CREATE TABLE labs (
  lab_id INT NOT NULL,
  lab_name VARCHAR(30) NOT NULL,
  lab_description VARCHAR(60),
  PRIMARY KEY (lab_id),
  UNIQUE (lab_name)
);

INSERT INTO labs 
    (lab_id, lab_name, lab_description) 
VALUES 
    (1,"Pod Netorking","Pod networking practical"),
    (2,"Network Policy","K8s network security");

SELECT * FROM labs;
+--------+----------------+--------------------------+
| lab_id | lab_name       | lab_description          |
+--------+----------------+--------------------------+
|      1 | Pod Netorking  | Pod networking practical |
|      2 | Network Policy | K8s network security     |
+--------+----------------+--------------------------+
2 rows in set (0.01 sec)

mysql> exit
Bye

kubectl delete pod mysql-68579b78bb-5rfzq

kubectl get pods

kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h mysql -ppassword

mysql>  USE labs_db;
mysql> SELECT * FROM labs;

mysql> exit
Bye
pod "mysql-client" deleted

https://github.com/RikKraanVantage/kubernetes-flask-mysql

aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 541301263746.dkr.ecr.us-east-2.amazonaws.com
docker build -t 541301263746.dkr.ecr.us-east-2.amazonaws.com/<registry-name>:<tag> .
docker push 541301263746.dkr.ecr.us-east-2.amazonaws.com/<registry-name>:<tag> 


aws ecr create-repository \
    --repository-name flask-api \
    --region us-east-2
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 541301263746.dkr.ecr.us-east-2.amazonaws.com
docker build -t 541301263746.dkr.ecr.us-east-2.amazonaws.com/flask-api .
docker push 541301263746.dkr.ecr.us-east-2.amazonaws.com/flask-api

aws ecr-public create-repository \
  --repository-name flaskapi \
  --region us-east-1 

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/c9i7u1v9

docker build -t pnivaggi/flaskback .
docker push pnivaggi/flaskback

kubectl run -it --rm --image=busybox --restart=Never bb-client -- wget -qO- -T 5 flask-service:5000

kubectl run -it --rm --image=busybox --restart=Never bb-client -- wget -qO- -T 5 flask-service:5000/blogs

show variables like '%char%';


kubectl delete deployment,svc mysql
kubectl delete pvc mysql-pv-claim
kubectl delete pv mysql-pv-volume

kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h mysql -ppassword --default-character-set=utf8

DROP DATABASE mydb
SHOW DATABASES;
show variables like '%char%';
#CREATE DATABASE mydb;
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
USE mydb;

CREATE TABLE blogs (
id INT NOT NULL,
blog_name VARCHAR(30) NOT NULL,
blog_content VARCHAR(60),
PRIMARY KEY (id),
UNIQUE (blog_name)
);

INSERT INTO blogs 
    (id, blog_name, blog_content) 
VALUES 
    (1,"K8s is fun","K8s allows to orchestrate application"),
    (2,"Network Policy","it provides network security using selector/label");

SELECT * FROM blogs;

kubectl run -it --rm --image=busybox --restart=Never bb-client -- wget -qO- -T 5 flask-service:5000
kubectl run -it --rm --image=busybox --restart=Never bb-client -- wget -qO- -T 5 flask-service:5000/blogs

CREATE TABLE blogs (
blog_name VARCHAR(30) NOT NULL,
blog_content VARCHAR(60),
PRIMARY KEY (blog_name)
);

INSERT INTO blogs 
    (blog_name, blog_content) 
VALUES 
    ("K8s is fun","K8s allows to orchestrate application"),
    ("Network Policy","it provides network security using selector/label");

## Deploy Pods

---

## Check Pods default communications

---

## Define and Apply Network Policy

---

## Check Network Policy