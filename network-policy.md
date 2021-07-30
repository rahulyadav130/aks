# Create the AKS cluster with calico network policies:
az aks create \
--resource-group myaksrg \
--name netPolicyCluster \
--node-count 3 \
--generate-ssh-keys \
--network-policy calico \
--network-plugin kubenet \
--service-principal "b692ba20-d946-4454-975e-a6699d0bf874" \
--client-secret "cbf13526-b9bf-4785-a16e-7ebff63ee4e4"


# Add the cluster into your current context:
az aks get-credentials --resource-group myaksrg --name netPolicyCluster

# Create YAML:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  labels:
    app: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: couchdb
        image: couchdb:2.3.0
        ports:
        - containerPort: 5984
---
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector:
    app: db
  ports:
    - name: db
      port: 15984
      targetPort: 5984
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: nodebrady
        image: mabenoit/nodebrady
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - name: api
      port: 8080
      targetPort: 3000
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - name: web
      port: 80
      targetPort: 80
  type: LoadBalancer

# Create multi-line deployment for myapp:
kubectl apply -f deployall.yaml

# List the services and pods in the cluster:
kubectl get po,svc

# Reach out to web service:
curl [web-svc-external-ip]

# Test connection from BusyBox pod via curl.
# Open a shell in the container:
kubectl run curlpod --image=radial/busyboxplus:curl --rm -it --generator=run-pod/v1

# Run 'curl' on the database service (once at the prompt for container):
curl http://db:15984

# Create a network policy to deny all ingress/egress:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# Apply the policy after pasting the above into a file named 'denyall.yaml':
kubectl apply -f denyall.yaml

# Test connection again after deny all policy has been applied:
curl --connect-timeout 2 [web-svc-external-ip]

# Create a network policy for the database pods:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-netpol
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
     - port: 5984
       protocol: TCP

# Apply the policy after pasting the above into a file named 'db-netpolicy.yaml':
kubectl apply -f db-netpolicy.yaml

# Create a network policy for the API pods:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-netpol
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
     - port: 3000
       protocol: TCP
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
     - port: 5984
       protocol: TCP
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
     - port: 53
       protocol: UDP

# Apply the policy after pasting the above into a file named 'api-netpolicy.yaml':
kubectl apply -f api-netpolicy.yaml

# Label the kube-system namespace to apply network policy to kube-dns pods:
kubectl label ns kube-system name=kube-system

# Test the network policy by applying the label 'app=api' to the curl pod.
# Open a shell in the container:
kubectl run curlpod --image=radial/busyboxplus:curl --labels app=api --rm -it --generator=run-pod/v1

# Run 'curl' on the database service (once at the prompt for container):
curl http://db:15984

# Try to connect to the web service (we should not be able to):
curl --connect-timeout 2 http://web:80

# Create a network policy for the web pods:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-netpol
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from: []
    ports:
     - port: 80
       protocol: TCP
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
     - port: 3000
       protocol: TCP
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
     - port: 53
       protocol: UDP

# Apply the policy after pasting the above into a file named 'web-netpolicy.yaml':
kubectl apply -f web-netpolicy.yaml

# Test connection from web pods to api pods.
# Open a shell in the container:
kubectl run curlpod --image=radial/busyboxplus:curl --labels app=web --rm -it --generator=run-pod/v1

# Run 'curl' on the API service (once at the prompt for container):
curl http://api:8080

# Try to connect to the database service (we should not be able to):
curl --connect-timeout 2 http://db:15984
