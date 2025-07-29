# Install Helm 

### From Apt (Debian/Ubuntu)
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
apt-get update
apt-get install helm
```
### Verify installation
```
helm version
```
# Install Web UI (Kubernetes Dashboard)
Note:-   
Kubernetes Dashboard supports only Helm-based installation currently as it is faster and gives us better control over all dependencies required by Dashboard to run.

The Dashboard UI is not deployed by default. To deploy it, run the following command:
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo list
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
kubectl get deploy -n kubernetes-dashboard
kubectl get pods -n kubernetes-dashboard
```
### Expose Dashboard
```
kubectl expose deployment kubernetes-dashboard-kong --name k8s-dash-svc --type NodePort --port 443 --target-port 8443 -n kubernetes-dashboard
kubectl get svc -n kubernetes-dashboard
```
check for port of k8s-dash-svc<br>
verify on browser > https://<node-ip\>:<port\>

# Creating sample user
To create a new user using the Service Account mechanism of Kubernetes, grant this user admin permissions and login to Dashboard using a bearer token tied to this user

### Creating a Service Account, ClusterRoleBinding and ServiceAccount
```
vi k8s-dash.yaml
```
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token
```
```
kubectl apply -f k8s-dash.yaml
```
### Execute the following command to get the token which is saved in the Secret
```
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d
```
### Accessing Dashboard
Now copy the token and paste it into the Enter token field on the login screen.

https://<node-ip\>:<port\>

Refer doc for more info https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md