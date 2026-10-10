# 🚀 Python Application Deployment Using Jenkins Pipeline

##  Project Overview

This project demonstrates the automated deployment of a **Python web application** using a **Jenkins Declarative Pipeline** and **AWS EC2**.

The application source code is maintained in GitHub, and Jenkins automates the process of fetching the source code, validating/building the application, and deploying it to an AWS EC2 instance using SSH.

The project helped me gain practical experience with **CI/CD automation, Jenkins Pipeline, AWS EC2, Linux, Git/GitHub, SSH-based deployment, and shell scripting**.

---

##  Architecture

```text
                 ┌──────────────────┐
                 │     Developer    │
                 └────────┬─────────┘
                          │
                          │ Push Code
                          ▼
                 ┌──────────────────┐
                 │     GitHub       │
                 │   Repository     │
                 └────────┬─────────┘
                          │
                          │ Checkout
                          ▼
                 ┌──────────────────┐
                 │     Jenkins      │
                 │     Pipeline     │
                 ├──────────────────┤
                 │ 1. Checkout      │
                 │ 2. Build/Verify  │
                 │ 3. Deploy        │
                 └────────┬─────────┘
                          │
                          │ SSH Deployment
                          ▼
                 ┌──────────────────┐
                 │     AWS EC2      │
                 │   Linux Server   │
                 ├──────────────────┤
                 │ Python Web App   │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │ Deployed Python  │
                 │   Application    │
                 └──────────────────┘
```

---

##  CI/CD Workflow

1. Developer pushes the Python application code to **GitHub**.
2. Jenkins starts the pipeline.
3. Jenkins checks out the source code from the GitHub repository.
4. The application is validated/built through the pipeline.
5. Jenkins uses configured SSH credentials to connect to the target EC2 instance.
6. The application files are transferred/deployed to the EC2 server.
7. The Python application is available on the target server.

---

##  Technologies Used

| Technology           | Purpose                       |
| -------------------- | ----------------------------- |
| **Python**           | Application development       |
| **Jenkins**          | CI/CD automation              |
| **Jenkins Pipeline** | Automated deployment workflow |
| **AWS EC2**          | Application hosting           |
| **Git**              | Version control               |
| **GitHub**           | Source code repository        |
| **Linux**            | Server environment            |
| **Shell Scripting**  | Deployment automation         |
| **SSH**              | Secure server connection      |

---

##  Project Structure

```text
python-jenkins-deployment/
│
├── Jenkinsfile
├── app/
│   ├── app.py
│   └── ...
│
├── requirements.txt
└── README.md
```

> The exact application files may vary depending on the Python application being deployed.

---

##  Jenkins Pipeline Stages

### 1. Checkout

Jenkins retrieves the latest application source code from the GitHub repository.

### 2. Build / Validation

The pipeline performs the required application validation and build steps before deployment.

### 3. Deployment

Jenkins connects to the AWS EC2 instance using SSH credentials and deploys the application to the configured application directory.

---

##  Jenkins Credentials

For secure deployment, Jenkins uses an SSH credential configured in **Jenkins Credentials Manager**.

Example configuration:

```text
Credential ID: ec2-target-key
```

The credential is referenced by the pipeline instead of exposing the private key directly inside the Jenkinsfile.

---

##  AWS EC2 Configuration

The target EC2 instance acts as the deployment server.

Required configuration includes:

* Running EC2 instance
* Linux-based operating system
* SSH access
* Appropriate Security Group rules
* Python environment
* Application deployment directory

![alt text](images/1.png)

---

#  Deployment Steps

# 1. Configure Jenkins

Install Jenkins on your Jenkins server if it is not already installed.

Open Jenkins:

```text
http://<JENKINS-SERVER-IP>:8080
```

Log in to the Jenkins dashboard.

![alt text](images/j-login.png)
---

# 2. Install Required Jenkins Plugins

From:

```text
Manage Jenkins
     ↓
Plugins
```

Make sure the required plugins are available, especially:

* Pipeline
* Git
* Credentials Binding
* SSH-related functionality as required by your deployment method

---

# 3. Add EC2 SSH Credentials to Jenkins

Go to:

```text
Manage Jenkins
     ↓
Credentials
     ↓
Global
     ↓
Add Credentials
```

Select:

```text
Kind: SSH Username with private key
```
![alt text](images/c1.png)

Configure:

```text
Username: ec2-user
ID: ec2-target-key
Private Key: <Your EC2 Private Key>
```
![alt text](images/c2.png)

The credential ID used in the Jenkinsfile should match:

```text
ec2-target-key
```
![alt text](images/c3.png)

###  Important

Do not hard-code your private SSH key inside the Jenkinsfile.

Jenkins Credentials Manager should be used to securely store credentials.

---

# 4. Create Jenkins Pipeline Job

From the Jenkins dashboard:

```text
New Item
```

Enter a project name.

Select:

```text
Pipeline
```

![alt text](images/c2.png)

---

# 5. Configure GitHub Repository

Inside the Pipeline configuration, use your GitHub repository as the source.

Example:

```text
https://github.com/iamankitajagtap/python-app-jenkins-pipeline.git
```

You can also keep the Jenkinsfile inside the GitHub repository and configure Jenkins to load the pipeline from SCM.

Recommended structure:

```text
GitHub Repository
        │
        ├── Jenkinsfile
        ├── app.py
        ├── requirements.txt
        └── other application files
```
![alt text](images/10.png)

---

# 6. Create the Jenkinsfile

Create a file named:

```text
Jenkinsfile
```

Example pipeline structure:

```groovy
pipeline {

    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        TARGET_IP = 'YOUR-EC2-PUBLIC-IP'
        CRED_ID   = 'ec2-target-key'
        APP_DIR   = '/home/ec2-user/python-static-site'
    }

    stages {

        stage('CHECKOUT') {
            steps {
                checkout scm
            }
        }

        stage('BUILD / VALIDATE') {
            steps {
                sh '''
                    python3 --version

                    if [ -f requirements.txt ]; then
                        echo "requirements.txt found"
                    fi

                    echo "Application validation completed"
                '''
            }
        }

        stage('DEPLOY') {
            steps {
                sshagent(credentials: [env.CRED_ID]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@$TARGET_IP "
                            mkdir -p $APP_DIR
                        "

                        rsync -avz --delete \
                            --exclude='.git' \
                            ./ ec2-user@$TARGET_IP:$APP_DIR/
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Python application deployed successfully!'
        }

        failure {
            echo 'Deployment failed. Check the console output.'
        }
    }
}
```

> Update the EC2 IP address, GitHub repository, application directory, and deployment commands according to your actual project.

---

---

# 7. Configure Pipeline from SCM

In Jenkins:

```text
Pipeline
   ↓
Definition
   ↓
Pipeline script from SCM
```

Select:

```text
SCM: Git
```

Enter your repository:

```text
https://github.com/iamankitajagtap/python-static-site.git
```
![alt text](images/4.png)

Select your branch:

```text
*/main
```
![alt text](images/5.png)

Set:

```text
Script Path: Jenkinsfile
```

Click:

```text
Save & Apply
```
![alt text](images/6.png)
---

# 8. Run the Jenkins Pipeline

Click:

```text
Build Now
```

Jenkins will execute the stages:

```text
CHECKOUT
    ↓
BUILD / VALIDATE
    ↓
DEPLOY
```

Monitor the process from:

```text
Build → Console Output
```
![alt text](images/7.png)
![alt text](images/8.png)

---

# 9. Verify Deployment on EC2

SSH into the EC2 instance:

```bash
ssh -i your-key.pem ec2-user@<EC2-PUBLIC-IP>
```

Navigate to the deployment directory:

```bash
cd /home/ec2-user/python-static-site
```

Check the files:

```bash
ls -la
```

You should see your application files.

![alt text](images/ec2-output.png)
---

# 10. Access the Application

Open your browser:

```text
http://<EC2-PUBLIC-IP>:<APPLICATION-PORT>
```

Example:

```text
http://3.xx.xx.xx:5000
```
![alt text](images/output.png)

---

##  Key Learning Outcomes

Through this project, I learned:

* How to create a **Jenkins Declarative Pipeline**
* How CI/CD pipelines automate deployments
* How to connect Jenkins with AWS EC2
* How to use Jenkins credentials securely
* How to perform SSH-based application deployment
* How to work with Linux servers
* How to automate deployment tasks using shell scripting
* How GitHub integrates with Jenkins workflows

---

##  Future Improvements

The project can be extended by adding:

* Automated testing
* Webhook-based pipeline triggers
* Docker containerization
* Environment-specific deployments
* Rollback mechanisms
* Monitoring and logging
* Deployment notifications

---

##  Conclusion

This project demonstrates a practical **CI/CD deployment workflow** using Jenkins and AWS EC2.

It provided hands-on experience in automating the journey from **source code in GitHub to a deployed Python application on AWS EC2**, strengthening my understanding of Jenkins, Linux, AWS, Git, and DevOps practices.

---

## 👩‍💻 Author

**Ankita Jagtap**

Cloud & DevOps Engineer