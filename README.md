# Ingress Controller with NGINX and Cert-Manager using DuckDNS

This guide is a way to set up an Ingress controller and cert-manager (using DuckDNS) to quickly and freely obtain an HTTPS URL pointing to our AKS cluster to expose our applications to the internet.

You can also find this guide in video form on my YouTube channel (javi__codes). More links to my other social networks at the end of the guide.

Let's get started:

## Prerequisites and Versions

- AKS cluster or EKS
- Helm 3
- Ingress-controller NGINX (via Helm)
- Cert-manager (via Helm)
- cert-manager DuckDNS webhook (via Helm)

## 1. Add Helm Repo for Ingress NGINX

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

## (2) Update repo 
```
helm repo update
```

## Install NGINX Ingress Controller with Helm


```
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress --create-namespace 
```

## Verify Running Pods

```
kubectl get pods -n ingress
```

```
NAME                                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-ingress-nginx-controller-74fb55cbd5-hjvr9   1/1     Running   0          41m
```

## Verify Ingress has a Public IP Assigned

```
kubectl get svc -n ingress
```

```
NAME                                               TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller             LoadBalancer   10.0.33.214    20.190.211.14   80:32321/TCP,443:30646/TCP   38m
```

## Deploy a Test Application

Now let's deploy an application running in a pod and a service to access the pods of this application. While it may seem unnecessary to deploy a single pod and a service for a single pod, remember that this pod can be deleted and replaced by another, which may not retain the same internal IP. A service avoids this issue by allowing us to use the same service to access the pod, regardless of how many pods there are.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: echo-app
  replicas: 2
  template:
    metadata:
      labels:
        app: echo-app
    spec:
      containers:
      - name: echo-app
        image: hashicorp/http-echo
        args:
        - "-text=Test application"
        ports:
        - containerPort: 5678
```

## Deploy a Service for the Test Application

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: echo-app
```

## Deploy an Ingress Resource

Now we will deploy an "ingress resource" that will tell our ingress controller to route traffic coming into our cluster via the public IP of the ingress controller (from step 5) to an internal service. Remember that our application's service does not have a public IP.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-echo
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: echo-svc
            port:
              number: 80

```

Here we are telling Kubernetes to create an ingress resource to send all incoming traffic through the ingress controller on port 80 to the echo-svc service on its port 80.

## Test the Setup

Now you can test accessing your site using the public IP from step 5.

Using a web browser: https://<Public-IP>
Using command line with curl: curl https://<Public-IP>


# Adding Certificates with Cert-Manager and DuckDNS


So far, so good, but our ingress has an IP (hard to remember) and everything is over HTTP, not encrypted. Let's add cert-manager, a solution that allows us to obtain TLS certificates for our web domains and rotate them automatically.

## Install Cert-Manager

```
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.2.0 --set 'extraArgs={--dns01-recursive-nameservers=8.8.8.8:53\,1.1.1.1:53}' --create-namespace --set installCRDs=true
```

After it finishes creating its resources, verify that the cert-manager pods are running correctly:

```
kubectl get pods -n cert-manager

```

```
NAME                                            READY   STATUS    RESTARTS   AGE
cert-manager-6c9b44dd95-59b6n                   1/1     Running   0          47m
cert-manager-cainjector-74459fcc56-6dfn8        1/1     Running   0          47m
cert-manager-webhook-c45b7ff-hrcnx              1/1     Running   0          47m
```

## We DuckDNS for temporary and free Domain

At this point, we are ready to request a TLS certificate for our site, BUT we need our own domain on the internet to point it to the public IP of our ingress controller (Step 5) so we can access our services using a domain name (or subdomain).

Another important point is that cert-manager only provides TLS certificates if it can verify that we own the domain we want to use. To do this, it uses two methods, but today we will discuss one called DNS-01. In this model, cert-manager will generate a TXT record in our DNS with a random value, then try to create this TXT record and read it. If it can read it, it means we have access to that domain (because cert-manager could create the TXT record). At that moment, cert-manager generates a certificate, deletes the TXT record, and stores the certificate (and its key) in a secret in our Kubernetes cluster.

Once this process is complete, we can create an ingress resource specifying that we want the incoming traffic from that domain (and a specific path) to use a certificate (stored in a secret) and redirect the traffic to a service (of our pod).

## Configure Your DuckDNS Account

Go to DuckDNS [https://www.duckdns.org/](https://www.duckdns.org/) and log in using one of the methods at the top of the page. Once done, you will see a TOKEN. This is what we will use in step 12 of this guide.

Also, further down the page, you will see a place to create a subdomain of duckdns.org and assign it an IP. That IP (IPv4) should be the IP of our ingress controller (from step 5 of this guide). After doing that, save the changes by clicking the button next to where you entered the IP (IPv4).

This tells DuckDNS to redirect all requests to that subdomain to the IP you entered.

## Deploy the DuckDNS Webhook Handler

We will deploy the DuckDNS webhook, which allows us to interact with DuckDNS. This webhook has a Helm chart, but it didn't work for me. The method that worked was cloning the repository and using its files.

Clone repo

```
git clone https://github.com/ebrianne/cert-manager-webhook-duckdns.git
```

Install from the cloned repository:
```
cd cert-manager-webhook-duckdns

helm install cert-manager-webhook-duckdns --namespace cert-manager --set duckdns.token='YOUR_DUCKDNS_TOKEN' --set clusterIssuer.production.create=true --set clusterIssuer.staging.create=true --set clusterIssuer.email='YOUR_EMAIL' --set logLevel=2 ./deploy/cert-manager-webhook-duckdns
```

At this point, there will be a new pod in our cert-manager namespace. We can see it with the following command:

```
kubectl get pods -n cert-manager
```

```
NAME                                            READY   STATUS    RESTARTS   AGE
cert-manager-webhook-duckdns-5cdbf66f47-kgt99   1/1     Running   0          56m
```

## ClusterIssuers and Cert-Manager Details

Cert-manager conceptually handles two types of ways to generate certificates: staging and production. The main difference is that production gives us a valid certificate for real-world use, one that browsers will recognize, while staging generates a certificate that browsers recognize as HTTPS but mark as "test."

The idea is to do all testing with staging because if we make mistakes and request too many incorrect certificates, we won't have problems. If we do the same in production, we might get banned for a certain period.

When we installed the DuckDNS webhook, we specified that we wanted to create the staging and production ClusterIssuers.

```
--set clusterIssuer.production.create=true --set clusterIssuer.staging.create=true
```

## Create an Ingress Resource Using the Staging Cluster Issuer

Create a file called staging-ingress.yaml with this content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-https-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: cert-manager-webhook-duckdns-staging
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - yoursubdomain.duckdns.org
    secretName: yoursubdomain-tls-secret-staging
  rules:
  - host: yoursubdomain.duckdns.org
    http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: echo-svc
            port:
              number: 80

```

In this example, my DuckDNS subdomain is yoursubdomain, and I am specifying that we use the cert-manager-webhook-duckdns-staging ClusterIssuer and store the certificate in a secret called yoursubdomain-tls-secret-staging. All traffic coming from yoursubdomain.duckdns.org over HTTPS will be sent to the echo-svc service on port 80.

The secret name can be anything, but it's good practice to make it descriptive and clear about its usage.

An important detail is that this ingress resource must be in the same namespace as the service to which we are redirecting traffic. The Ingress CONTROLLER can (and should) be in a different namespace, as should cert-manager.

Apply it with

```
kubectl apply -f staging-ingress.yaml
```

## Verify Certificate Creation Process


Now, if in our namespace where we deployed the ingress resource and the pods/services we run a `kubectl get challenge`, we will see something like this:
```
NAME                                                        STATE     DOMAIN                       AGE
yoursubdomain-tls-secret-staging-6lmxj-668717679-4070204345   pending   yoursubdomain.duckdns.org      4s
```

This is the process cert-manager uses to generate the TXT record in DuckDNS and verify that we own that domain/subdomain. Once verified, this challenge disappears, and a certificate is generated and stored in a secret named as specified in the ingress resource (yoursubdomain-tls-secret-staging in our case).

If we check the status of our certificate while the challenge exists, we'll see something like this:

```
NAME                             READY   SECRET                           AGE
yoursubdomain-tls-secret-staging   False    yoursubdomain-tls-secret-staging   7m15s
```
And once the process is finished and the challenge disappears, we see it like this:

```
NAME                             READY   SECRET                           AGE
yoursubdomain-tls-secret-staging   True    yoursubdomain-tls-secret-staging   7m15s
```

At this point, we can verify access to our domain yoursubdomain.duckdns.org using a browser or curl.

```
curl https://yoursubdomain.duckdns.org/
curl: (60) schannel: SEC_E_UNTRUSTED_ROOT (0x80090325) - The certificate chain was issued by an authority that is not trusted.
```

This is correct; we have a certificate, but it is a "staging" certificate, not a "production" certificate.



## Adjust Ingress Resource to Request a Production Certificate

Now, create a new file called production-ingress.yaml with the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-https-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: cert-manager-webhook-duckdns-production
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - yoursubdomain.duckdns.org
    secretName: yoursubdomain-tls-secret-production
  rules:
  - host: yoursubdomain.duckdns.org
    http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: echo-svc
            port:
              number: 80
```

```
kubectl apply -f production-ingress.yaml
```

Once done, follow the same steps as before to check everything is set up correctly. Finally, verify with your browser or curl that your HTTPS site is working. If everything went well, you should see something like this:

```
curl https://yoursubdomain.duckdns.org/
Test application
```

## Troubleshooting

Here are my recommendations for troubleshooting if something goes wrong:

a) Check the logs of the cert-manager webhook pods. The DuckDNS webhook pod will show all the steps cert-manager takes against DuckDNS, useful if your TOKEN is incorrect or if DuckDNS isn't responding correctly.

b) Check the logs of the cert-manager pod named cert-manager-XXXX. This shows information on cert-manager's activities, such as creating or modifying the secret where the certificate is stored.

c) Check the logs of the ingress-controller. Here you'll see incoming requests, their origin IP, and any errors, helping identify misconfigurations.

d) Use a tool to verify that the IP you entered in DuckDNS points to the public IP of your ingress controller. A tool I use is https://digwebinterface.com/.


