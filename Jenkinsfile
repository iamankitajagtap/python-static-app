
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        TARGET_IP = '34.207.96.215'
        CRED_ID   = 'ec2-target-key'
        APP_DIR   = '/home/ec2-user/python-static-site'
        APP_PORT  = '5000'
        REPO_URL  = 'https://github.com/iamankitajagtap/python-static-app.git'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out Python application repository...'
                git branch: 'main', url: "${REPO_URL}"
            }
        }

        stage('Install Python') {
            steps {
                sh '''
                    set -e
                    python3 --version
                    python3 -m pip --version
                '''
            }
        }

        stage('Create Virtual Environment') {
            steps {
                sh '''
                    set -e
                    python3 -m venv venv
                    ./venv/bin/python -m pip install --upgrade pip
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -e
                    ./venv/bin/pip install -r requirements.txt
                    ./venv/bin/pip install gunicorn
                '''
            }
        }

        stage('Build and Validate') {
            steps {
                sh '''
                    set -e
                    ./venv/bin/python -m py_compile app.py
                    ./venv/bin/python -c \
                      "from app import app; print('Flask application imported successfully')"
                    echo 'Python application validation successful.'
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
                        set -e

                        SSH_OPTS="-i $SSH_KEY -o StrictHostKeyChecking=no -o BatchMode=yes"

                        echo 'Creating application directory...'

                        ssh $SSH_OPTS "$SSH_USER@$TARGET_IP" \
                            "mkdir -p '$APP_DIR'"

                        echo 'Copying application files...'

                        scp $SSH_OPTS app.py requirements.txt \
                            "$SSH_USER@$TARGET_IP:$APP_DIR/"

                        if [ -d templates ]; then
                            scp $SSH_OPTS -r templates \
                                "$SSH_USER@$TARGET_IP:$APP_DIR/"
                        fi

                        if [ -d static ]; then
                            scp $SSH_OPTS -r static \
                                "$SSH_USER@$TARGET_IP:$APP_DIR/"
                        fi

                        echo 'Application files copied successfully.'
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
                        set -e

                        ssh -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            -o BatchMode=yes \
                            "$SSH_USER@$TARGET_IP" \
                            "APP_DIR='$APP_DIR' bash -s" <<'REMOTE_SCRIPT'

set -e

sudo dnf install -y python3

cd "$APP_DIR"

python3 -m venv venv
./venv/bin/python -m pip install --upgrade pip
./venv/bin/pip install -r requirements.txt
./venv/bin/pip install gunicorn

echo 'Target environment setup completed.'

REMOTE_SCRIPT
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
                        set -e

                        ssh -i "$SSH_KEY" \
                            -o StrictHostKeyChecking=no \
                            -o BatchMode=yes \
                            "$SSH_USER@$TARGET_IP" \
                            "APP_DIR='$APP_DIR' APP_PORT='$APP_PORT' bash -s" <<'REMOTE_SCRIPT'

set -e

cd "$APP_DIR"

# Stop the previous Gunicorn process if its PID file exists.
if [ -f gunicorn.pid ]; then
    OLD_PID=$(cat gunicorn.pid)

    if [[ "$OLD_PID" =~ ^[0-9]+$ ]] &&
       kill -0 "$OLD_PID" 2>/dev/null; then
        kill "$OLD_PID" || true

        for i in {1..10}; do
            kill -0 "$OLD_PID" 2>/dev/null || break
            sleep 1
        done
    fi

    rm -f gunicorn.pid
fi

# Start Gunicorn in the background.
nohup ./venv/bin/gunicorn \
    --bind "0.0.0.0:$APP_PORT" \
    --workers 2 \
    --pid gunicorn.pid \
    app:app \
    >> app.log 2>&1 < /dev/null &

# Wait for the Flask application to respond.
HEALTHY=0

for i in {1..15}; do
    if curl -fsS "http://127.0.0.1:$APP_PORT/" -o /dev/null; then
        HEALTHY=1
        echo 'Application health check passed.'
        break
    fi

    sleep 2
done

if [ "$HEALTHY" -ne 1 ]; then
    echo 'Application health check failed.'
    tail -50 app.log || true
    exit 1
fi

echo 'Gunicorn application is running.'
tail -20 app.log

REMOTE_SCRIPT
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Python application deployed successfully.'
            echo "Application URL: http://${TARGET_IP}:${APP_PORT}"
        }

        failure {
            echo 'Python application deployment failed. Check Console Output.'
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
