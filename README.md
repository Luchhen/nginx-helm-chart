Reference link: https://gitlab.progressoft.io/progressoft/recipes/baremetal/harbor 

https://gitlab.progressoft.io/progressoft/recipes/baremetal/argocd 

  

Workstation: 

- Create two VM in oracle using nat adapter 

- Port forwarded and accessible via host IP.  

- One master and One worker using k3s 

  

Installation steps for harbor: 

1. Download offline installer from "https://github.com/goharbor/harbor/releases/tag/v2.5.6" 

2. Extract harbor-offline-installer-v2.5.6.tgz using tar zxvf command 

3. Under /harbor which is extracted from point 2, copy template file (harbor.yml.tmpl to harbor.yml ) then update harbor.yml by needed values like IP , HTTPs TLS or not , proxy  

4. Under /harbor/install.sh 

5.  after it finish you can access Harbor Using web browser : 

URL : Yourdomin.com  / server IP 
defult user: admin 
defulat password: Harbor12345 

 

Replication setup Steps 

1- Using web browser access : Yourdomin.com and login. 

2- From left panel, click on Registries --> new endpoint. 

fill your source harbor info as attached screenshot : 

 

3- From left panel, click on Replications --> new replication rule. 

fill your source harbor info as attached screenshot : 

 

Test sample deployment on k3s using local harbor 

1- you need to connect k3s on all nodes with this harbor by add below file : 

iips@k3s-test-server1:/etc/rancher/k3s$ cat registries.yaml 
Then you must restart k3s on all node : 

systemctl restart k3s  

or 

reboot the server.  

2- create below sample deployment ( check image part : image: 10.0.2.15:8080/sierra/prompt-check:v269 ). 

Installation of AgroCD 

Step 1 — Create ArgoCD namespace: 

kubectl create namespace argocd 

Step 2 — Install ArgoCD: 

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 

Step 4 — Check all pods are ready: 

kubectl get pods -n argocd 

# Should see these all Running: 

# argocd-server 

# argocd-repo-server 

# argocd-application-controller 

# argocd-dex-server 

# argocd-redis 

 

Step 5 — Expose ArgoCD UI (since you're on k3s/VM, use NodePort): 

kubectl patch svc argocd-server -n argocd \ 

  -p '{"spec": {"type": "NodePort"}}' 

Step 6 — Get the NodePort assigned: 

kubectl get svc argocd-server -n argocd 

# Look for PORT(S) like 80:3XXXX/TCP, 443:3XXXX/TCP 

 

Step 7 — Get initial admin password:  

kubectl -n argocd get secret argocd-initial-admin-secret  

-o jsonpath="{.data.password}" | base64 -d && echo 

``` 

Then access UI from your host browser: 

 

Create a simple nginx app with Helm Chart in Github: 

Initialize and push to GitHub: 

cd nginx-argocd-app 

git init 

git add . 

git commit -m "Initial nginx helm chart" 

git branch -M main 

git remote add origin https://github.com/Luchhen/nginx-helm-chart.git 

git push -u origin main 

 

Build and push image to Harbor: 

cd app/ 

docker build -t 10.0.2.15:8080/library/nginx-app:latest . 

docker login http://10.0.2.15:8080 -u admin -p Harbor12345 

docker push 10.0.2.15:8080/library/nginx-app:latest  

Create Harbor imagePullSecret in k3s: 

kubectl create secret docker-registry harbor-secret \  

--docker-server=10.0.2.15:8080 \ 

--docker-username=admin \  

--docker-password=Harbor12345 \  

--namespace=default 

Connect ArgoCD to your GitHub repo via UI: 

Create ArgoCD Application: 

Applications → New App 

App Name: nginx-app  

Project: default  

Sync Policy: Automatic  

Repo URL:  

Path: helm/nginx-app  

Cluster: https://kubernetes.default.svc  

Namespace: default https://github.com/Luchhen/nginx-helm-chart.git 

Helm Values: values.yaml 

Verify deployment: 

kubectl get pods -n default 

kubectl get svc nginx-app 

Access from host browser 

http://192.168.1.68:<Nodeport> 

GitHub (Helm Chart) → ArgoCD → pulls manifest → k3s deploys pod → pulls image from Harbor 

Want to test the GitOps flow? Try this: 

Edit your index.html 

vim app/index.html 

Change the message to "Hello from GitOps v2! 🎯" 

Rebuild and push new image 

docker build -t 10.0.2.15:8080/library/nginx-app:v2 .  

docker push 10.0.2.15:8080/library/nginx-app:v2 

Update values.yaml tag to v2 

git commit & push → ArgoCD auto-syncs → new pod deploys! 

 

FULL CI/CD pipeline Using GitHub Actions: 

 

 

Install and start as a service: 

sudo ./svc.sh install 
sudo ./svc.sh start 

 

 

Create ci.yaml 

Push workflow to GitHub: 

mkdir -p .github/workflows 

create ci.yaml with content above 

git add .github/workflows/ci.yaml  

git commit -m "ci: add github actions workflow"  

git push origin main 

 
**What happens on every push to main: ** 
  

You push code change to app/  

↓  

GitHub Actions triggers on self-hosted VM runner  

↓ 

 Runner builds Docker image locally  

↓ 

 Runner pushes image to Harbor (10.0.2.15:8080) ✅ 

 ↓  

Runner updates values.yaml with new image tag  

↓  

Runner pushes values.yaml back to GitHub  

↓  

ArgoCD detects values.yaml changed  

↓  

ArgoCD syncs → k3s pulls new image from Harbor 

↓ New pod running with latest code! ✅ 

 

 

 

 

 
