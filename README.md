<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Wordpress on IBM Kubernetes Service (IKS)](#wordpress-on-ibm-kubernetes-service-iks)
  - [Included Components](#included-components)
  - [Objectives](#objectives)
  - [Deployment Option 1: Deploy a Mysql container to use with Wordpress](#deployment-option-1-deploy-a-mysql-container-to-use-with-wordpress)
    - [Step 1: Setup MySQL Secrets](#step-1-setup-mysql-secrets)
    - [Step 2: Create Persistent volumes claims](#step-2-create-persistent-volumes-claims)
    - [Step 3: Create Deployments and Services for WordPress and MySQL](#step-3-create-deployments-and-services-for-wordpress-and-mysql)
    - [Step 4: Retrieve Ingress subdomain and configure domain DNS](#step-4-retrieve-ingress-subdomain-and-configure-domain-dns)
  - [Deployment Option 2: Using IBM CLoud Compose for MySQL as backend](#deployment-option-2-using-ibm-cloud-compose-for-mysql-as-backend)
    - [Step 1: Deplpy Compose for Mysql](#step-1-deplpy-compose-for-mysql)
    - [Step 2: Retrieve MySQL connection information](#step-2-retrieve-mysql-connection-information)
    - [Step 3: Create Deployments and Services for WordPress](#step-3-create-deployments-and-services-for-wordpress)
    - [Step 4: Retrieve Ingress subdomain and configure domain DNS](#step-4-retrieve-ingress-subdomain-and-configure-domain-dns-1)
  - [Troubleshooting](#troubleshooting)
  - [References](#references)
  - [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Wordpress on IBM Kubernetes Service (IKS)

This is my fork of the official Scalable Wordpress example [here](https://github.com/IBM/Scalable-WordPress-deployment-on-Kubernetes). I am going to outline 2 options for this deployment:

 1. [Run mysql in a container](#deployment-option-1-deploy-a-mysql-container-to-use-with-wordpress) 
 2. [Use the IBM Cloud Compose for Mysql service](#deployment-option-2-using-ibm-cloud-compose-for-mysql-as-backend-work-in-progress)

Both of these options require that you be running a standard (non-free tier) Kubernetes cluster. 

## Included Components
- [WordPress (Latest)](https://hub.docker.com/_/wordpress/)
- [MySQL (5.6)](https://hub.docker.com/_/mysql/)
- [Kubernetes Clusters](https://console.ng.bluemix.net/docs/containers/cs_ov.html#cs_ov)
- [IBM Cloud Compose for MySQL](https://console.ng.bluemix.net/catalog/services/compose-for-mysql)
- [IBM Cloud Container Service](https://console.ng.bluemix.net/catalog/?taxonomyNavigation=apps&category=containers)

## Objectives
This scenario provides instructions for the following tasks:

- Create persistent volume claims to store data for Wordpress and Mysql.
- Create a secret to protect sensitive data.
- Create and deploy the WordPress frontend with one or more pods.
- Create and deploy the MySQL database (either in a container or using IBM Cloud MySQL as backend).

## Deployment Option 1: Deploy a Mysql container to use with Wordpress

### Step 1: Setup MySQL Secrets

Create a new file called `password.txt` in the same directory and put your desired MySQL password inside `password.txt` (Could be any string with ASCII characters).

```bash
echo "YOUR_PASSWORD" | tee password.txt
```

We need to make sure `password.txt` does not have any trailing newline. Use the following command to remove possible newlines.

```bash
tr -d '\n' <password.txt >.strippedpassword.txt && mv .strippedpassword.txt password.txt
```

### Step 2: Create Persistent volumes claims

To save your data beyond the lifecycle of a Kubernetes pod, you will want to create persistent volumes for your MySQL and Wordpress applications to attach to. In this example I am using the `ibmc-file-retain-silver` storageclass. You can find the supported storageclasses by issuing the command `kubectl get storageclasses`.

```
kubectl create -f wordpress-mysql-pvc.yaml
```

This will take a few moments as the File storage is provisioned and bound to our cluster. You can check the progress by issing the command `kubectl get pvc -l app=wordpress`. When both volume claims show their status as `bound` you are good to move on to the next section.

### Step 3: Create Deployments and Services for WordPress and MySQL

We will be storing our Mysql password as a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) and creating our deployment and service for mysql. 

```bash
kubectl create secret generic mysql-pass --from-file=password.txt
kubectl create -f mysql-deployment.yaml
```

We are going to expose our Wordpress site using an [Ingress controller](https://console.bluemix.net/docs/containers/cs_ingress.html#ingress). Before you create the Wordpress deployment you will need to update the `wordpress-deployment.yaml` and substitute your domain for `example.com`. 

```
sed -i 's|example.com|yourdomain.com|' wordpress-deployment.yaml
kubectl create -f wordpress-deployment.yaml
```

**Note**: if you are running these commands on a Mac use the following sed command instead:

```
sed -i '' 's|example.com|yourdomain.com|' wordpress-deployment.yaml
```

When all your pods are running, run the following commands to check your pod names.

```bash
kubectl get pods -l app=wordpress
```

This should return a list of pods from the kubernetes cluster.

```bash
NAME                             READY     STATUS             RESTARTS   AGE
wordpress-66fff7ddb5-9hcpd       1/1       Running            0          1m
wordpress-66fff7ddb5-b4zm2       1/1       Running            0          1m
wordpress-66fff7ddb5-kgbp5       1/1       Running            3          1m
wordpress-mysql-cf9449df-6s8b2   1/1       Running            0          4m
```

### Step 4: Retrieve Ingress subdomain and configure domain DNS

In order to use our domain with the Wordpress site we need point create a CNAME recoed to point to our Ingress Subdomain. To retrieve the ingress subdomain run the following command:

```
ibmcloud ks cluster-get <cluster_name_or_ID>
```

Once you have the Ingress subdomain create the needed CNAME record at your domain registar. For example I am using the domain `wp.tinylab.info` so at my registar I create a CNAME record to point `wp` to `rt-k8s.us-east.containers.appdomain.cloud`

![DNS CNAME Record](images/dns_record.png)

Once the DNS record has propagated you can visit your domain and you should be presented with the Wordpress initial configuration page. 

![Wordpress Initial Configuration](images/wordpress_configuration.png)

## Deployment Option 2: Using IBM CLoud Compose for MySQL as backend

### Step 1: Deplpy Compose for Mysql 

You can provision Compose for MySQL in IBM Cloud via the [IBM Cloud portal](https://console.ng.bluemix.net/catalog/services/compose-for-mysql) at or via the IBM Cloud cli using the following command:

```
ibmcloud service create compose-for-mysql Standard <your service name>
```

### Step 2: Retrieve MySQL connection information

Go to Service credentials and view your credentials. Your MySQL hostname, port, user, and password are under your credential uri and it should look like this (in my screenshot the password has been obscured, but it is the string just after the username).

![MySQL Connection Information](images/mysql_connection_info.png)

### Step 3: Create Deployments and Services for WordPress

Modify your `wordpress-deployment.yaml` file, change `WORDPRESS_DB_HOST`'s value to your MySQL hostname and port (i.e. `value: <hostname>:<port>`), `WORDPRESS_DB_USER`'s value to your MySQL user, and `WORDPRESS_DB_PASSWORD`'s value to your MySQL password.

And the environment variables should look like this

```yaml
    spec:
      containers:
      - image: wordpress:4.7.3-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: sl-us-south-1-portal.19.dblayer.com:50883
        - name: WORDPRESS_DB_USER
          value: admin
        - name: WORDPRESS_DB_PASSWORD
          value: xxxxxxxxxxxxxxxxxx
```

After you modified the `wordpress-deployment.yaml`, run the following commands to deploy WordPress.

```
sed -i 's|example.com|yourdomain.com|' wordpress-deployment.yaml
kubectl create -f wordpress-deployment.yaml
```

**Note**: if you are running these commands on a Mac use the following sed command instead:

```
sed -i '' 's|example.com|yourdomain.com|' wordpress-deployment.yaml
```

When all your pods are running, run the following commands to check your pod names.

```bash
kubectl get pods -l app=wordpress
```

This should return a list of pods from the kubernetes cluster.

```bash
NAME                             READY     STATUS    RESTARTS   AGE
wordpress-66fff7ddb5-9hcpd       1/1       Running   0          12h
wordpress-66fff7ddb5-b4zm2       1/1       Running   0          12h
wordpress-66fff7ddb5-kgbp5       1/1       Running   5          12h
```

### Step 4: Retrieve Ingress subdomain and configure domain DNS

In order to use our domain with the Wordpress site we need point create a CNAME recoed to point to our Ingress Subdomain. To retrieve the ingress subdomain run the following command:

```
ibmcloud ks cluster-get <cluster_name_or_ID>
```

Once you have the Ingress subdomain create the needed CNAME record at your domain registar. For example I am using the domain `wp.tinylab.info` so at my registar I create a CNAME record to point `wp` to `rt-k8s.us-east.containers.appdomain.cloud`

![DNS CNAME Record](images/dns_record.png)

Once the DNS record has propagated you can visit your domain and you should be presented with the Wordpress initial configuration page. 

![Wordpress Initial Configuration](images/wordpress_configuration.png)

## Troubleshooting

If you accidentally created a password with newlines and you can not authorize your MySQL service, you can delete your current secret using

```bash
kubectl delete secret mysql-pass
```

If you want to delete your services, deployments, and persistent volume claim, you can run
```bash
kubectl delete deployment,service,pvc,ingress -l app=wordpress
```

If you want to delete your persistent volume, you can run the following commands
```bash
kubectl delete -f wordpress-mysql-pvc.yaml
```

If WordPress is taking a long time, you can debug it by inspecting the logs
```bash
kubectl get pods -l app=wordpress
kubectl logs [wordpress pod name]
```

## References
- This WordPress example is based on Kubernetes's open source example [mysql-wordpress-pd](https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd) at https://github.com/kubernetes/kubernetes/tree/master/examples/mysql-wordpress-pd.


## License
[Apache 2.0](LICENSE)
