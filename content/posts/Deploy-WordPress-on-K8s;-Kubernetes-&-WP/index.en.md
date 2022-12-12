---
title: "Deploy WordPress on K8s; Kubernetes & WP"
date: 2022-12-10T17:55:28+08:00
lastmod: 2022-12-10T17:55:28+08:00
draft: false
description: "How to Deploy WordPress on K8s Kubernetes"
resources:
- name: "featured-image"
  src: "featured--image.jpg"

tags: ["wordpress","kubernetes","K8s", "ingress"]
categories: ["kubernetes","wordpress"]

twemoji: false
lightgallery: true
---

How to Deploy WordPress on K8s Kubernetes with SSL

<!--more-->
## Solution Overview

In this post, you will learn to deploy WordPress on k8s (Kubernetes) WordPress is a popular content management system (CMS) for creating websites easily. By using PHP and MySQL to support WordPress in building a content management system, we can create many websites, blogs, and other website-based applications by learning how to deploy WordPress on Kubernetes nodes.
    
### Preparation

This blog assumes little prior experience with Kubernetes, but does expect you to have kubectl and helm installed. You should have access to your cluster available to the console user and familiarity with text editing

We’ll be using AKS for the example environment.

1. You will need kubectl working in your environment. See here for complete instructions.

2. You will need the LiteSpeed Ingress Controller installed and operational. See here for complete instructions. You can skip the Making  HTTPS Work step for now as we will be doing that later.

3. You will need cert-manager installed and operational. See here for complete instructions. Perform the steps up to and including Creating a ClusterIssuer. The steps below that will define the Ingress.

4. You will need a real domain name that is available on the internet, and you will need to configure your environment to publish it.

### Create some sample definitions

In addition to the previously mentioned preparation, we recommend the following sample definitions for simplicity:

``kubectl create ns woo``


Create a number of secrets to hold some useful definitions. You can change these and just note to change their YAML references below:

* The database name which we’ll name multitenant-wp
* The admin database password which we’ll name mysqlpwd
* The database user which we’ll name userwp
* The database user password which we’ll name pwdwp

```
kubectl create -n woo secret generic mysql-database --from-literal=database=multitenant_wp
kubectl create -n woo secret generic mysql-password --from-literal=password=mysqlpwd
kubectl create -n woo secret generic mysql-user --from-literal=username=userwp
kubectl create -n woo secret generic mysql-user-password --from-literal=password=pwdwp
```

## Install ingress Nginx

I will use helm to install the ingress controller Nginx on the nodes. Therefore, you can visit the helm website for installation.

```
$ helm upgrade --install ingress-nginx ingress-nginx \
>   --repo https://kubernetes.github.io/ingress-nginx \
>   --namespace ingress-nginx --create-namespace 

```

Congratulations! You have successfully installed the ingress controller Nginx– the first step is already down. Now, the next step is pointing the IP from ingress to your domain.

Then, you can use the domain wordpress.you_domain.com to be the Public IP Ingress!

## Install Cert-Manager

We use cert-manager to manage and create certificate SSL. Using cert-manager, we can create “SSL letsencrypt.”

Letsencrypt is reliable to our website become secure connections. We use jetstack repository to install cert-manager. If you install jetstack, the first step is to update the jetstack repository.

```
$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories
```

After that, the first time you update the jetstack repository using helm, you can update helm repository to install the cert-manager.

By now, you have already prepared helm and repository to install cert-manager. At this step, the cert-manager requires CRD to run properly.

```
$ helm install \
>   cert-manager jetstack/cert-manager \
>   --namespace cert-manager \
>   --create-namespace \
>   --version v1.7.1 \
>   --set installCRDs=true
```

## Create the persistent volumes
You will need persistent volume definitions. To create them use the text editor of your choice and create the file named 1-pv.yaml:

```Yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  namespace: woo
spec:
  storageClassName: do-block-storage
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/var/lib/mysql"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: woo
spec:
  storageClassName: do-block-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
  namespace: woo
spec:
  storageClassName: do-block-storage
  capacity: 
    storage: 30Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/var/www"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pv-claim
  namespace: woo
spec:
  storageClassName: do-block-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
---
```
If you changed your namespace you’ll need to update the namespace definitions above.
To create the volumes, apply the 1-pv.yaml file:

``kubectl apply -f 1-pv.yaml``

## Install MySQL for WordPress

To install WordPress, you’ll need to install MySQL first and use the persistent volumes you created above. Create the file named: 2-mysql.yaml:

```Yaml
apiVersion: v1
kind: Service
metadata: 
  name: mysql-wp
  namespace: woo
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-wp
  namespace: woo
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
      - image: mysql:latest
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-password
              key: password
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mysql-user
              key: username
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-user-password
              key: password
        - name: MYSQL_DATABASE
          valueFrom:
            secretKeyRef:
              name: mysql-database
              key: database
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
```

If you changed the namespace or the MySQL user or passwords secret titles you’ll need to update the definitions above.

To install MySQL apply the 2-mysql.yaml file:

``kubectl apply -f 2-mysql.yaml``

## Install WordPress

To install WordPress itself, create the file named 3-wordpress.yaml:

```Yaml
apiVersion: v1
kind: Service
metadata: 
  name: wordpress
  namespace: woo
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: woo
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: web
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: web
    spec:
      containers:
      - image: wordpress:php8.1-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql-wp:3306
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-user-password
              key: password
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-user
              key: username
        - name: WORDPRESS_DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-database
              key: database
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pv-claim
```

If you changed the namespace or the MySQL user or passwords secret titles you’ll need to update the definitions above.

To install WordPress apply the 3.mysql.yaml file:

``kubectl apply -f 3-wordpress.yaml``

## Expose WordPress & Configure letsencrypt 

In this part of this post, let’s create an SSL letsencrypt issuer on the Kubernetes nodes. The issuer will help us generate and renew letsencrypt certificates. Moreover, letsencrypt is useful to make a website secure. It provides open-source SSL certificates to create HTTPS on every website in the world. Then, let’s create a file called wp_production_issuer.yaml and fill this below.

```Yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: 
  name: wp-prod-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: email@punitkashyap.in
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

After that, deploy wp_production_issuer.yaml on the Kubernetes nodes.

``kubectl apply -f wp_production_issuer.yaml``

## Configure ingress HTTPS

Since you have already deployed the issuer letsencrypt certificates to Kubernetes nodes, create an ingress so that WordPress can use the HTTPS. I have used domain wordpress.example.net in the ingress configuration which will use SSL letsencrypt so that the wordpress.example.net domain can be accessed using HTTPS. Firstly, we pointed the IP from the ingress controller Nginx to the destination domain to run the WordPress service. After that, create wordpress_ingress.yaml for configuring ingress WordPress.

```Yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wordpress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "wp-prod-issuer"
spec:
  rules:
  - host: wordpress.example.net
    http:
     paths:
     - path: "/"
       pathType: Prefix
       backend:
         service:
           name: wordpress
           port:
             number: 80
  tls:
  - hosts:
    - wordpress.example.net
    secretName: wordpress-tls
```

Deploy wordpress_ingress.yaml to Kubernetes nodes.

``kubectl apply -f wordpress_ingress.yaml -n namespace woo``

## Configure WordPress

How about that? You have successfully configured WordPress and deployed it into Kubernetes. Now, it’s time for us to open the domain previously configured on ingress WordPress.

Use the domain wordpress.example.net for this. 

## Deleting everything and putting it all back

f you need to delete everything to start over or simply to clean it up after testing, you should delete the entities in the reverse order you created them:

**Delete All Deployement**
```Yaml
kubectl delete -n woo -f database-pv.yaml 
kubectl delete -n woo -f mysql.yaml
kubectl delete -n woo -f wordpress.yaml
kubectl delete -n woo -f wordpress_ingress.yaml


kubectl delete -n woo secret mysql-database
kubectl delete -n woo secret mysql-password
kubectl delete -n woo secret mysql-user
kubectl delete -n woo secret mysql-user-password
```
I do not recommend that you delete the namespace as it may enter a Terminating condition it may not be able to get out of.

``kubectl delete namespace woo``

## Conclusion

WordPress is a powerful content management system for us as landing pages and blogs, apart from being open source, we can also build WordPress anywhere and deploy WordPress on Kubernetes nodes. Isn’t that cool?

If we need a WordPress site with a high load, we can build WordPress on Kubernetes by adjusting the scaling of pods. Aside you can use the Horizontal Pod Autoscaler HPA to scale horizontally based on CPU and even other metrics. Kubernetes is very helpful for building highly loaded applications. If you don’t know how to set up Kubernetes on a Linux server, you can check this article Kubernetes: [k8s setup and run apps](https://sesamedisk.com/kubernetes-nodes-setting-up-k8s-multi-node-run-apps/) before learning how to deploy WordPress on Kubernetes nodes.
