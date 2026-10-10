pipeline {

    agent any

    environment {

        TARGET_IP = '34.207.96.215'

        CRED_ID = 'ec2-target-key'

        APP_DIR = '/home/ec2-user/python-static-site'

    }

    stages {

        stage('Checkout') {

            steps {

                echo 'Cloning Python application...'

                git branch: 'main',
                    url: 'https://github.com/iamankitajagtap/python-static-app.git'

            }
        }


        stage('Install Python') {

            steps {

                sh '''
                    sudo yum install -y python3

                    python3 --version

                    pip3 --version
                '''

            }
        }


        stage('Create Virtual Environment') {

            steps {

                sh '''
                    rm -rf venv

                    python3 -m venv venv

                    source venv/bin/activate

                    pip install --upgrade pip
                '''

            }
        }


        stage('Install Dependencies') {

            steps {

                sh '''
                    source venv/bin/activate

                    pip install -r requirements.txt
                '''

            }
        }


        stage('Build / Validate') {

            steps {

                sh '''
                    source venv/bin/activate

                    python3 -m py_compile app.py

                    echo "Python application validation successful"
                '''

            }
        }


        stage('Deploy to Target Server') {

            steps {

                withCredentials([

                    sshUserPrivateKey(

                        credentialsId: "${CRED_ID}",

                        keyFileVariable: 'SSH_KEY',

                        usernameVariable: 'SSH_USER'

                    )

                ]) {

                    sh '''

                        echo "Creating application directory..."

                        ssh -i $SSH_KEY \
                            -o StrictHostKeyChecking=no \
                            $SSH_USER@$TARGET_IP \
                            "mkdir -p $APP_DIR"


                        echo "Copying application to target server..."

                        scp -i $SSH_KEY \
                            -o StrictHostKeyChecking=no \
                            -r app.py requirements.txt templates static \
                            $SSH_USER@$TARGET_IP:$APP_DIR/


                        echo "Application copied successfully."

                    '''

                }

            }
        }


        stage('Setup Application on Target') {

            steps {

                withCredentials([

                    sshUserPrivateKey(

                        credentialsId: "${CRED_ID}",

                        keyFileVariable: 'SSH_KEY',

                        usernameVariable: 'SSH_USER'

                    )

                ]) {

                    sh '''

                        ssh -i $SSH_KEY \
                            -o StrictHostKeyChecking=no \
                            $SSH_USER@$TARGET_IP "

                            sudo yum install -y python3

                            cd $APP_DIR

                            python3 -m venv venv

                            source venv/bin/activate

                            pip install --upgrade pip

                            pip install -r requirements.txt

                            echo 'Python dependencies installed.'

                        "

                    '''

                }

            }
        }


        stage('Start Application') {

            steps {

                withCredentials([

                    sshUserPrivateKey(

                        credentialsId: "${CRED_ID}",

                        keyFileVariable: 'SSH_KEY',

                        usernameVariable: 'SSH_USER'

                    )

                ]) {

                    sh '''

                        ssh -i $SSH_KEY \
                            -o StrictHostKeyChecking=no \
                            $SSH_USER@$TARGET_IP "

                            cd $APP_DIR

                            source venv/bin/activate

                            pkill -f 'gunicorn.*app:app' || true

                            nohup gunicorn \
                                --bind 0.0.0.0:5000 \
                                --workers 2 \
                                app:app \
                                > app.log 2>&1 &

                            echo 'Application started successfully.'

                        "

                    '''

                }

            }
        }

    }


    post {

        success {

            echo '======================================'

            echo ' Python Application Deployed Successfully '

            echo '======================================'

        }

        failure {

            echo 'Python application deployment failed.'

        }

    }

}
