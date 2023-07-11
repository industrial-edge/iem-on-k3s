# Deploying Industrial Edge Management on K3s Cluster
To deploy the Industrial Edge Management on a K3s cluster, you must perform the following tasks:

* [Install K3s Cluster](/docs/Installation.md)
* [Create an IEM instance in the Industrial Edge Hub (IEH)](/docs/Deployment.md#create-an-iem-instance-in-the-industrial-edge-hub)
* [Download and install the IE Provisioning CLI](/docs/Deployment.md#download-nad-install-the-ie-provisioning-cli)
* [Install the IEM on the K3s Cluster](/docs/Deployment.md#install-the-iem)


## Create an IEM Instance in the Industrial Edge Hub
1. Log into the Industrial Edge Hub.
2. Navigate to `IEM Instances` and click `Create New IEM Instance`.
3. Enter the name and, optionally, the description of the IEM instance.
4. Click `Save`.
   The IEM instance has been created.
5. Click the download icon to download the configuration file of the newly created IEM instance.

![Create IEM instance in IE Hub](/docs/graphics/create-iem.png)

## Download nad Install the IE Provisioning CLI
1. Log into the Industrial Edge Hub.
2. Navigate to `Download Software`.
3. In the `Industrial Edge Provisioning CLI - Ubuntu` tile, click `Download`
4. Make sure the file is executable:
```bash
chmod +x ieprovision
```
5. To execute the IE Provisioning CLI, install the binary:
```bash
sudo install ieprovision /usr/local/bin/
```

## Install the IEM on K3s cluster
### Configure kubeconfig files
The ieprovision command-line tool uses kubeconfig files to find the information it needs to choose a cluster and communicate with API server of a cluster.  
Create `.kube` directory in home directory and copy kubeconfig file to this directory `~/.kube`:  
```bash
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```
Change Access rights to kubeconfig file to use ieprovision cli without `sudo` privileges:
```bash
sudo chown $USER ~/.kube/config
```
Change Access rights to kubeconfig file to use kubectl without sudo privileges:
```bash
sudo chown $USER /etc/rancher/k3s/k3s.yaml
```
### Install the IEM
Before proceeding with the installation, ensure that the [general requirements of IEM 2.0](https://industrial-edge.code.siemens.io/product/documentation/admin-operator-documentation/k-8-s-cluster/general-reqs.html) are satisfied. Following guide describes the minimal installation steps for fully functional IEM deployment. For detailed information on the installation process and available options refer to the official [IEM documentation](https://industrial-edge.code.siemens.io/product/documentation/admin-operator-documentation/k-8-s-cluster/index.html).

#### Create certificate chain for TLS communication
To setup the IEM SSL-Certificates are required. You can refer to [this documentation](https://industrial-edge.code.siemens.io/product/documentation/admin-operator-documentation/k-8-s-cluster/minikube/deployment-with-minikube.html#ingress-controller-activation) as an example how to create certificates for your IEM instance.

#### Prepare the ieprovision template file
Use the `template` command ot create a quickstart configuration file:
```bash
ieprovision template > template.yaml
```
The template file looks similar to the following example:
```yaml
central-auth:
    keycloak:
        initialUser:
            # 
            # email of the initial keycloak user
            email: iem.user@siemens.com
            # 
            # first name of the initial keycloak user
            firstName: iem
            # 
            # last name of the initial keycloak user
            lastName: user
            # 
            # username of the initial keycloak user
            username: iem_user
global:
    # 
    # Application Secret Key
    applicationSecretKey: c4fa6d29-b581-49d7-a929-baa6881e63e3
    # 
    # Password for Customer Realm Admin
    customerAdminPassword: SOf33u2q$#So
    # 
    # Password for database user
    databaseUserPassword: ""
    # 
    # Hostname for accessing the IEM
    hostname: ""
    # 
    # Password for IAM administrator
    iamAdminPassword: Vy97#P@mq3%K
    # 
    # Password for IAM Auth Proxy
    iamAuthProxyClientSecret: ""
    # 
    # Client Secret for IAM SDK
    iamSdkClientSecret: ""
    # 
    # Password for IEM administrator
    iemAdminPassword: K*iU#3*4Eo4A
```
The template file contains default values which must be changed by the operator.

The FQDN (fully qualified domain name) of the IEM must be used for the `hostname` in the template, e.g. `iem.edge.com`

Extend the template with storage class definition:
```yaml
global:
    #
    # Storage class which will be used for creating Persistent Volume Claims for IEM Core Services like Docker Registry Service
    storageClass: local-path
    #
    # Storage class which will be used for creating Persistent Volume Claim for Postgress database
    storageClassPg: local-path
```
K3s comes with Rancher's [Local Path](https://docs.k3s.io/storage) (storageClassName: `local-path`) Provisioner.

The installation of the IEM is executed by using the configuration file of the IEM instance and by using the completed template file:
```bash
ieprovision install <configuration-file.json> --values template.yaml --set global.certChain="$(cat ./certificateChain.crt | base64 -w 0)" --set global.gateway.ingress.enabled=false
```

| Parameter                      | Description                                                                                                                                                                                                                                                                                                                                       | Value                         |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| global.certChain               | Add the Root and Intermediate CA Certificates of the Entrypoint (Ingress, Loadbalancer) to the IEM System. This certificates will be stored in the certificate store of the devices, to establish a secure connection to the IEM.                                                                                                                 | "$(cat ./certificateChain.crt | base64 -w 0)" |
| global.gateway.ingress.enabled | Creation of ingress rule, which will point to the IE Gateway. By default ieprovision tool creates rule based on nginx which is not the ingress controller shipped with K3s cluster. [Traefik](https://traefik.io/) is the default deployed ingress controller in K3s. When using Traefik set parameter `global.gateway.ingress.enabled` to false. | false                         |

After the installation is completed the console output should look as following:
```bash
NOTES:
CHART NAME: application-management-service
CHART VERSION: v1.1.85
APP VERSION: v1.1.85

** Please be patient while the chart is being deployed **

IEM: https://iem.edge.siemens.com/

NAMESPACE: iem

KEYCLOAK_ADMIN: admin
KEYCLOAK_ADMIN_PASSWORD: XE!Mq4HzsbGg

KEYCLOAK_USER: iem_user
KEYCLOAK_USER_PASSWORD: C&Obe2BvONbw

KEYCLOAK_CUSTOMER_REALM_ADMIN: customer_admin
KEYCLOAK_CUSTOMER_REALM_ADMIN_PASSWORD: GDzBNq@K46vU
CUSTOMER REALM ADMIN LOGIN: https://iem.edge.siemens.com/auth/admin/customer/console/

ACCOUNT_CONSOLE: https://iem.edge.siemens.com/auth/realms/customer/account
Helm chart deployed
```
Please make sure to save this output in a secure location as contains the initial login credentials for the IEM.

#### Deploy Ingress of the IEM on the K3s Cluster
The Industrial Edge offers an enhanced API gateway built on [Kong OSS](https://konghq.com/install#kong-communit), serving as the primary access point to the system. It handles authentication, verifies session cookies, and generates JSON web tokens for internal API requests. In a Kubernetes (K8s) environment, there are various deployment options available for the API gateway. By default, it is deployed as a ClusterIP service type. For initial configuration of the Traefik ingress rule, it is essential to obtain the service name of the Industrial Edge Gateway.

![IEM Gateway and Ingress controller architecture](/docs/graphics/gateway-ingress.png)
*IEM Gateway and Ingress controller architecture*

Get service name of Industrial Edge Gateway:
```bash
kubectl get service -n iem | grep gateway-proxy
```
Output:
```bash
ie801b-gateway-proxy            ClusterIP   10.43.227.122   <none>        80/TCP,443/TCP              4d1h
```

##### Create Secret for Ingress Rule
**Option 1:**
Create `secret.yaml` file:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: iemcert
  namespace: iem
data:
  # base64 encoded cert & key
  tls.crt: <certificate>
  tls.key: <privatekey>
```
Insert certificate and private key base64 encoded.  
To encode certificate (not certificate chain) and private key to base64 use the following command and copy output to `secret.yaml`:
```bash
cat <certificate.crt> | base64 -w 0
```
```bash
cat <privatekey.key> | base6e -w 0
```
Deploy the secret:
```bash
kubectl apply -f secret.yaml
```

**Option 2:**
Deploy secret:
```bash
kubectl -n iem create secret tls iemcert --cert=<certificate.crt> --key=<privatekey.key>
```

##### Create Ingress Rule
Create `ingress.yaml` file:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: iem-ingress
  namespace: iem
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    #
    # Hostname for accessing the IEM e.g. iem.siemens.com
    - <Hostname> 
    secretName: iemcert
  rules:
  #
  # Hostname for accessing the IEM e.g. iem.siemens.com
  - host: <Hostname>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            #
            # Servicename of Industrial Edge Gateway
            name: <ie801b-gateway-proxy>
            port:
              name: kong-proxy
```
Insert values for `hosts` and `host` according to the value set for `hostname` in the template file and service name of Industrial Edge Gateway in `ingress.yaml`.

Finally, deploy the Ingress rule:
```bash
kubectl apply -f ingress.yaml
```
