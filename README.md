
# Deploying TLS Certificates for Local Development and Production using Kubernetes, Cert-Manager, mkcert

## 1. Prerequisites
Ensure you have the following installed on your system:
- **Minikube** ([Installation Guide](https://minikube.sigs.k8s.io/docs/start/))
- **Kubectl** ([Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/))
- **Nginx Ingress Controller**
- **mkcert** for local SSL certificate generation
- **OpenSSL** for generating self-signed certificates (optional)

## 2. Start Minikube and Enable Ingress
Start Minikube and enable the **Nginx Ingress Controller**:

```sh
minikube start
minikube addons enable ingress
```
Verify the Ingress Controller is running:
```sh
kubectl get pods -n kube-system | grep ingress-nginx
```
3. Generate a Self-Signed Certificate
```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout tls.key -out tls.crt \
-subj "/CN=myapp.local/O=myapp.local"
```
3.1Alternatively, if you want to use mkcert to create a self-signed certificate, you can run:
```sh

choco install mkcert
```
3.2 Create a Local Certificate Authority (CA)
```sh
mkcert -install
```
This installs a local CA, which is trusted by your system.

3.3 Generate an SSL Certificate for Your Local Domain
```sh
mkcert todoapi2.local.com
```
This command generates two files:

todoapi2.local.com.pem (certificate)
todoapi2.local.com-key.pem (private key)

4. Create a Kubernetes TLS Secret
Store the generated SSL certificate in a Kubernetes secret:
```sh
kubectl create secret tls magda-local-cert-tls --namespace default \
  --cert=todoapi2.local.com.pem \
  --key=todoapi2.local.com-key.pem
  ```
Verify that the secret is created:

```sh
kubectl get secrets -n default
```
5. Create Kubernetes Resources
You need to define three essential resources for your application: the Deployment, Service, and Ingress.

5.1 Create the Deployment Resource
The Deployment resource will ensure that the application runs as a pod in Kubernetes, creating a container based on the andrews011/todo-api image.

Create a file deployment.yaml with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-app
  labels:
    app: todo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-app
  template:
    metadata:
      labels:
        app: todo-app
    spec:
      containers:
        - name: todo-app
          image: andrews011/todo-api
          ports:
            - containerPort: 80
```
This deployment ensures that your todo-app will be running in the cluster with one replica.

## 5.2 Create the Service Resource
The Service resource exposes your app on a specific port, allowing external access within the cluster.

Create a file service.yaml with the following content:

```yaml
Copy
Edit
apiVersion: v1
kind: Service
metadata:
  name: todo-service
spec:
  selector:
    app: todo-app  # This should match the label of your application pods
  ports:
    - protocol: TCP
      port: 80   # External port that the service exposes
      targetPort: 80  # Internal port of the container where the app is listening
  type: ClusterIP
```
5.3 Create the Ingress Resource
Next, configure the Ingress resource to use the TLS secret for secure HTTPS traffic.

Create a file ingress.yaml with the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: local-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - todoapi2.local.com
    secretName: magda-local-cert-tls
  rules:
  - host: todoapi2.local.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: todo-service
            port:
              number: 80
```              
6. Apply the Kubernetes Resources
To apply the Deployment, Service, and Ingress resources:

Apply the Deployment:

```sh
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```
Verify that the Deployment, Service, and Ingress are created:

```sh
kubectl get deployments -n default
kubectl get services -n default
kubectl get ingress -n default
```
Expected output for the Ingress:

NAME            CLASS   HOSTS                ADDRESS   PORTS     AGE
```lua
local-ingress   nginx   todoapi2.local.com             80, 443   10h
If the ADDRESS field is empty, restart Minikube:
```
minikube tunnel
7. Verify HTTPS Configuration
7.1 Check Certificate in Browser
Open your browser and navigate to:


https://todoapi2.local.com

Ensure that the certificate is properly recognized and the connection is secure.

7.2 Validate with OpenSSL
Run the following command:

```sh
openssl s_client -connect todoapi2.local.com:443 -servername todoapi2.local.com
```

You should see:

```yaml
Certificate chain
 0 s:CN=todoapi2.local.com, O=Magda
   i:CN=todoapi2.local.com, O=Magda
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jan 29 08:26:29 2025 GMT; NotAfter: Jan 29 08:26:29 2026 GMT
```   
## 8. Troubleshooting
Issue: Browser Shows "Not Secure" Warning
Ensure cert manager or mkcert is installed correctly by running:
Restart your browser and clear cache.

## Issue: Ingress Address Field is Empty
Start Minikube tunnel:

```sh
minikube tunnel
```
## Issue: Certificate Not Found in Kubernetes
Check if the secret is created:

```sh
kubectl get secrets -n default
```
If missing, recreate it and apply the Ingress YAML again.

9. Conclusion
You have successfully set up HTTPS with Nginx Ingress on Minikube using mkcert or OpenSSL, and exposed your todo-app via the todo-service. Your local application (todoapi2.local.com) is now accessible over a secure HTTPS connection.

🚀 Happy coding!
