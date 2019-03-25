# Securing Aks with WAF 
While an Ingress service can offer load balancing and OWASP capabilities, it is essential to secure your AKS cluster from the outside by way of adding a WAF. Options for WAF on Azure are using Application Gateway or any custom 3rd Party WAF on top of Azure IaaS. Given below is a sample that uses Azure Kubernetes Service (AKS) with an Application Gateway as a WAF and a NGinx ingress controller

The solution is depicted in the diagram below

![AKS with App G with Ingress](/AKS%20with%20WAF.png)

This sample builds on the resources created in the workshop [https://aksworkshop.io/]. Please refer to this for applicaiton related files and source code. 

## Step 1 - Build AKS Cluster

We will deploy an AKS Cluster without HTTPApplicationRouting setup and advanced networking to enable integration with other services on Azure and on-premises. For advanced networking - create a virtual network with a subnet for AKS and create another subnet for Application Gateway. 

## Step 2 - Deploy App

Deploy the following pods and services in the workshop. 

### Pods
a. MongoDB database
b. Capture-Order API
c. Frontend

### Services
i. MongoDB DB
ii. Capture-Order Service - choose type as Cluster IP instead of Load Balancer. Sample yaml provided below

```
apiVersion: v1
kind: Service
metadata:
  name: capture-order
spec:
  selector:
    app: captureorder
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
  ```
  
  iii. Frontend Service - The existing sample also deploys a Cluster IP service - so keep it as is
  
  iv. DO NOT configure the HTTP Application Routing add on or the ingress service as provided in the sample. Go to next step instead
  
  ## Step 3
  
  Deploy an NGINX Ingress Controller with a private IP address. While we will use helm we will need to override configuration to specify internal load balancer. Copy yaml provided below into a file named nginx-internal-ingress.yaml
  
  ```
  controller:
  service:
    loadBalancerIP: 10.240.0.42
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
  ```
  
  Now you can setup the ingress controller using helm as indicated below
  
  ```
  helm install stable/nginx-ingress \
    --namespace kube-system \
    -f internal-ingress.yaml \
    --set controller.replicaCount=2
  ```

## Step 4 Deploy Ingress Rules

Configure the required ingress rules. Looking at the example - we will put the frontend in the default path and create a /api path that redirects to the Capture-Order api. 

Given below is the yaml for the same

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: frontendv2-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: frontend
          servicePort: 80
      - path: /api
        backend:
          serviceName: capture-order
          servicePort: 80
```

Deploy this in the cluster by running the command

  ```
    kubectl apply -f yringress.yaml
  ```

## Step 5 Deploy Application Gateway

You can deploy the application gateway in the App Gateway subnet. Sample provided below. If you want to do it through the portal - refer [[https://docs.microsoft.com/en-us/azure/application-gateway/quick-create-portal] here]

```
az network application-gateway create \
  --name myAppGateway \
  --location eastus \
  --resource-group akschallenge \
  --capacity 2 \
  --sku Standard_Medium \
  --http-settings-cookie-based-affinity Enabled \
  --public-ip-address myAGPublicIPAddress \
  --vnet-name <AKS VNET Name> \
  --subnet <subnet created for App gateway> \
  --servers "<address1>" "<address2>" 
```

Addresses are the addresses assigned to the NGINX controller service. You can get the same by running

  ```
    kubectl get svc --namespace kube-system
  ```
  and selecting the IP Addresses assigned to the NGINX controller service. 
  

## Step 6

Test your application. Browse to the IP address of the Application Gateway or the dns url, as well as with a suffix of /api/v1/order. 
  
  


