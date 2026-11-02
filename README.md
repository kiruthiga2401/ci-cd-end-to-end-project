# 🚀 CI/CD Pipeline with Jenkins, Docker, SonarQube and Argo CD

I built this project to learn how code goes from a `git push` all the way to a running app on Kubernetes, with no manual steps in between. It takes a Java Spring Boot app, builds and tests it with Jenkins, checks the code quality with SonarQube, packs it into a Docker image, and lets Argo CD deploy it to Kubernetes.

🛠️ **Tools used:** Git, Maven, Jenkins, SonarQube, Docker, Helm, Argo CD, Kubernetes, AWS EC2

## 🔄 How it works

<img width="847" height="427" alt="Screenshot 2026-11-02 200215" src="https://github.com/user-attachments/assets/66a1033b-e9c0-41ec-8812-85ce5b99e4fe" />


1. 📥 Jenkins pulls the latest code from GitHub.
2. 🔨 Maven builds the app and runs the unit tests.
3. 🔍 SonarQube checks the code quality.
4. 🐳 The app is packed into a Docker image and pushed to Docker Hub.
5. 📝 The new image tag is written into the manifest file in Git.
6. ☸️ Argo CD notices the change and deploys the new version to Kubernetes.

---

## 🏗️ Part A: Getting Jenkins up and running

### 1. ☁️ Launch an EC2 instance
Start an Ubuntu instance from the AWS Console. Pick something with at least 2 CPUs and 2 GB RAM, because SonarQube needs it later. Then connect to it using its public IP:

```bash
ssh -i your-key.pem ubuntu@<ec2-public-ip>
```

### 2. ☕ Install Java and Jenkins
Jenkins needs Java, so install that first:

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

Then add the Jenkins repository and install it:

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

### 3. 🔓 Open the ports and log in
AWS blocks incoming traffic by default, so Jenkins won't load until you open its port. Go to **EC2 > Instances > Security > Security group** and add inbound rules for:

- 🟢 **8080** for Jenkins
- 🟢 **8010** for the demo app
- 🟢 **9000** for SonarQube

Now open `http://<ec2-public-ip>:8080` in your browser. Get the unlock password with:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

After logging in, go to **Manage Jenkins > Plugins** and install these, then restart Jenkins: Docker Pipeline, Git, Maven Integration, Pipeline, SonarQube Scanner and Kubernetes Continuous Deploy. 🔌

---

## 📦 Part B: The app and the tools

### 4. 🧪 Try the app locally first
Before automating anything, make sure the app actually builds and runs:

```bash
git clone https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero.git
cd Jenkins-Zero-To-Hero/java-maven-sonar-argocd-helm-k8s/spring-boot-app
sudo apt install maven -y
mvn clean package
```

Once Docker is installed (next step), run it as a container and open `http://<ec2-public-ip>:8010`:

```bash
docker build -t ultimate-cicd-pipeline:v1 .
docker run -d -p 8010:8080 -t ultimate-cicd-pipeline:v1
```

💡 **Tip:** running the JAR directly uses port 8080, which Jenkins already uses, so Docker is the easier route.

### 5. 🐳 Install Docker
```bash
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
sudo systemctl restart docker
```

Those `usermod` lines let Jenkins and the `ubuntu` user run Docker without `sudo`. Restart Jenkins afterwards at `http://<ec2-public-ip>:8080/restart` 🔁

<img width="1887" height="808" alt="Screenshot 2026-11-02 135708" src="https://github.com/user-attachments/assets/5b59ecac-7ce6-45fb-8204-7c594257186e" />

### 6. 📊 Install SonarQube
SonarQube needs Java 17+ and about 2 GB of RAM. It also refuses to run as root, so we create a separate user for it.

```bash
sudo apt install unzip -y
sudo adduser sonarqube
cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip
sudo unzip sonarqube-10.4.1.88267.zip
sudo mv sonarqube-10.4.1.88267 sonarqube
sudo chown -R sonarqube:sonarqube /opt/sonarqube
sudo chmod -R 775 /opt/sonarqube

su - sonarqube
cd /opt/sonarqube/bin/linux-x86-64
./sonar.sh start
```

Open `http://<ec2-public-ip>:9000`. The default login is `admin` / `admin`. After logging in, create a token under **My Account > Security**. You'll need it in a moment. 🔑

---

<img width="1917" height="946" alt="Screenshot 2026-11-02 135536" src="https://github.com/user-attachments/assets/3e3c7365-d99a-42c0-b92c-9ed92b0a3296" />


## ⚙️ Part C: The Jenkins pipeline

### 7. 🧩 Create the pipeline job
In Jenkins, click **New Item**, choose **Pipeline**, and set the definition to **Pipeline script from SCM** with Git. Add your repository URL, the branch and `Jenkinsfile` as the script path, then save.

### 8. 🔐 Add your credentials
Go to **Manage Jenkins > Credentials > Global > Add Credentials** and add three:

| ID | Type | What it's for |
|----|------|---------------|
| 🔑 `sonar-token` | Secret text | Logging in to SonarQube |
| 🐳 `docker-cred` | Username & Password | Pushing images to Docker Hub |
| 🐙 `github` | Personal Access Token | Updating the manifest in Git |

### 9. ▶️ Run it
Click **Build Now** and watch the **Console Output**. When you see **SUCCESS** ✅, the whole chain worked: checkout, Maven build, unit tests, SonarQube analysis, Docker image build, push to Docker Hub, and manifest update.

⚠️ If a stage fails, the console output usually tells you why. Wrong credentials and Docker permissions are the most common causes.

---

## ☸️ Part D: Deploying with Argo CD

### 10. 🎯 Set up Argo CD
Install it on your Kubernetes cluster, check that the pods are running, and expose the server:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n argocd
```

Grab the admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Log in to the Argo CD UI and click **New App**. Point it to your manifests repo and path, use `https://kubernetes.default.svc` as the cluster, and turn on **Automatic** sync.

### 11. 🔗 Connect Argo CD to Jenkins
Add the Argo CD token to Jenkins credentials (for example `argocd-token`) and add a deploy stage to your Jenkinsfile that uses it.

### 12. 🎉 See it live
From now on, every push triggers the same flow: Jenkins builds a new image, pushes it to Docker Hub and updates the manifest. Argo CD sees the change and deploys it. Check it with:

```bash
kubectl get pods
kubectl get svc
```
<img width="1912" height="1007" alt="Screenshot 2026-11-02 154400" src="https://github.com/user-attachments/assets/bce90add-e251-4c7c-a2d8-938f160d4bbd" />


Then open `http://<node-ip>:<service-nodeport>` in your browser and your app is running. 🥳

<img width="1387" height="425" alt="Screenshot 2026-11-02 153329" src="https://github.com/user-attachments/assets/f45b82eb-32a1-462f-be93-14105d2763a8" />


---

