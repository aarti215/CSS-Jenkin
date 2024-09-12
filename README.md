# ${\color{red} \textbf{Host CSS Template}}$

${\color{purple} \textbf{TOOLS}}$

- Git
- GitHub
- Docker
- K8s
- Jenkins

${\color{green} \textbf{1. Download CSS template and Extract it}}$

${\color{green} \textbf{2. Add that All extracted file in newly created Repository}}$

${\color{green} \textbf{3. Create Dockerfile }}$

${\color{green} \textbf{4. Create a Instance}}$
- ubuntu
- t2.large
- port : 22, 80, 8080 

${\color{green} \textbf{5. Install jenkins and Host it}}$
````
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl daemon-reload
sudo systemctl enable jenkins
sudo systemctl start jenkins
````
- cat /var/lib/jenkins/secrets/initialAdminPassword
- Host `server-ip:8080` and Starting Jenkins Server.

  
${\color{green} \textbf{6. Install Plugins}}$
- stage view
- Docker (all)
- Kubernetes
- AWS credentials

${\color{green} \textbf{7. Add Credentials}}$
- Docker Credential (Username and password seperated)
- AWS Credentials (Access key & Secret key)


${\color{green} \textbf{8. Install Docker }}$
````
sudo apt-get install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
````
````
sudo chmod 666 /var/run/docker.sock
#sudo gpasswd -aG jenkins docker
sudo gpasswd -a jenkins docker
sudo chown jenkins /var/run/docker.sock
````

${\color{green} \textbf{9. (EKS)Create a Cluster and NodeGroup with only one Node}}$

${\color{green} \textbf{10. Paste below commands on jenkins server to configure kubernetes}}$
````
snap install aws-cli --classic
````
````
aws configure
````
NOTE : Add access key and Secret key.
````
aws eks update-kubeconfig --region ap-south-1 --name <cluster-name>
````
````
aws eks update-kubeconfig --region ap-south-1 --name my-cluster --kubeconfig /tmp/config        #ap-south-1(Region_of_Cluster) & my-cluster(cluster_Name)
````
````
chown jenkins:jenkins /tmp/config
````

${\color{green} \textbf{11. Install Kubectl command on jenkins server}}$
````
sudo curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
sudo chmod +x kubectl
      mkdir -p ~/.local/bin
      mv ./kubectl ~/.local/bin/kubectl
      # and then append (or prepend) ~/.local/bin to $PATH
````
````
sudo kubectl cluster-info
````
${\color{green} \textbf{12. Create deployment file}}$
`k8s-pipeline.yml`  
````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: css-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      name: pipeline-tag
  template:
    metadata:
      name: my-css
      labels:
        name: pipeline-tag
    spec: 
      containers:
      - name: docker-jenkins
        image: guru6910/css.c1      #DockerHub_Image
        ports:
        - containerPort: 80
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: css-service
spec:
  selector:
    name: pipeline-tag      #This tag should match the labels in the deployment
  ports:
  - name: http
    targetPort: 80
    port: 80
    protocol: TCP
  type: LoadBalancer
````

${\color{green} \textbf{13. Create Pipeline}}$

ADD WEBHOOK
- Poll SCM
- `* * * * *`
- pipeline script from SCM
- Git
- https://github.com/guru6910/CSS-jenkins.git
- `jenkinsfile`
````
pipeline {   
    agent any
    environment {
        DOCKERHUB_REPO = 'guru6910/css.c1'
    }
    stages {
        stage('git-checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/guru6910/DEMO.git'
            }
        }
        
        stage('docker-build') {
            steps {
                sh 'docker build -t guru6910/css.c1:${BUILD_NUMBER} .'
            }
        }
        
        stage('docker-push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-cred', passwordVariable: 'passwd', usernameVariable: 'uname')]) {
                    sh "docker login -u ${env.uname} -p ${env.passwd}"
                    sh 'docker push guru6910/css.c1:${BUILD_NUMBER}'
                }
            }
        }
        
        stage('deploy-k8s') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-cred', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        sh """
                    aws eks update-kubeconfig --name my-cluster --region ap-south-1 --kubeconfig /tmp/config
                    kubectl apply -f k8s-pipeline.yml --kubeconfig=/tmp/config
                    kubectl set image deployment/css-deployment docker-jenkins=guru6910/css.c1:${BUILD_NUMBER} --kubeconfig=/tmp/config
                    """
                    }
                }
            }
        }
    }
}
````
````
kubectl set image deployment/css-deployment docker-jenkins=sohampatil08/devops-tool-jenkins-pipeline:${env.BUILD_NUMBER} --kubeconfig=/tmp/config
````
This command updates the docker-jenkins container image in the deployment to a specific version of the <sohampatil08/devops-tool-jenkins-pipeline> image

${\color{green} \textbf{14. Hit on Build Now, You will get result of Build stages:}}$

![image](https://github.com/user-attachments/assets/7beecc0b-3d08-4d0c-90f8-02c1f635a1e0)


${\color{green} \textbf{15. Get the DNS of LoadBalancer from jenkins Terminal}}$

![image](https://github.com/user-attachments/assets/b0eab705-3704-4c3a-8e1b-6648b68ba0b3)

${\color{green} \textbf{16. Hit the DNS on Browser to watch Deployed Application:}}$

![image](https://github.com/user-attachments/assets/c7a385e3-c8b6-4ec8-9727-a048ad1df3fc)
