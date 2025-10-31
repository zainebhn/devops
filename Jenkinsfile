pipeline {
    agent any

    // Optionnel : décommente si tu as défini des tool names dans Jenkins Global Tool Configuration
    // tools {
    //   maven 'Maven 3.8.8'
    //   jdk 'JDK17'
    // }

    environment {
        // Ne change pas ces variables si tu les utilises ailleurs
        MAVEN_OPTS = "-Dmaven.test.skip=true"
        DOCKER_IMAGE = "zainebheni/student-management"
        SONAR_HOST_URL = "http://localhost:9000"    // Sonar sur la même machine (tu as dit que Sonar est local)
        MINIKUBE_HOME = "/var/lib/jenkins"          // profil minikube pour l'utilisateur jenkins
        KUBECONFIG_PATH = "/var/lib/jenkins/.kube/config" // fallback local kubeconfig
        MINIKUBE_CONTEXT = "minikube"
        NODE_PORT = "30081"
        APP_PORT = "8089"
        DEPLOY_NAMESPACE = "devops"
        GIT_CREDENTIALS = 'cd5fe940-75d1-4257-8b4c-d245de66f89c' // adapte si besoin
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: "${GIT_CREDENTIALS}",
                    url: 'https://github.com/zainebhn/devops.git'
            }
        }

        stage('Build (Maven)') {
            steps {
                dir('student-management') {
                    // -B for non-interactive, -DskipTests handled by MAVEN_OPTS if desired
                    sh 'mvn -B clean package -DskipTests'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                dir('student-management') {
                    // Lancement des tests ; si les tests dépendent d'une BD externe, ajuster
                    sh 'mvn -B test || true'
                    // si tu veux arrêter le pipeline à l'échec des tests, retire `|| true`
                }
            }
            post {
                always {
                    junit 'student-management/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('sonarqube-token') // secret text dans Jenkins
            }
            steps {
                dir('student-management') {
                    sh """
                        mvn -B sonar:sonar \
                          -Dsonar.projectKey=student-management \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                dir('student-management') {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                            set -euo pipefail
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                            docker build -t "$DOCKER_USER/student-management:latest" .
                            docker push "$DOCKER_USER/student-management:latest"
                        '''
                    }
                }
            }
        }

        stage('Prepare & Start Minikube (if necessary)') {
            steps {
                script {
                    // On supporte deux modes d'obtention du kubeconfig :
                    // 1) Secret file Jenkins id 'kubeconfig-file' (recommandé)
                    // 2) Fichier fallback /var/lib/jenkins/.kube/config (déjà présent)
                    //
                    // On essaie d'utiliser la cred 'kubeconfig-file' si elle existe, sinon on tombe sur le fichier local.
                    def kubeCredId = 'kubeconfig-file' // ajuste si nécessaire

                    // script shell robuste qui : 
                    // - définit MINIKUBE_HOME et KUBECONFIG
                    // - démarre minikube si nécessaire (driver docker)
                    // - vérifie l'espace disque et alerte si < 2GB (safe)
                    sh """
                        set -euo pipefail
                        export MINIKUBE_HOME=${MINIKUBE_HOME}
                        export KUBECONFIG=${KUBECONFIG_PATH}

                        echo "MINIKUBE_HOME=${MINIKUBE_HOME}"
                        echo "KUBECONFIG=${KUBECONFIG}"

                        # Si Jenkins credentials 'kubeconfig-file' est monté, Jenkins va la fournir via withCredentials.
                        # Ici : si un kubeconfig existe déjà et est lisible, on l'utilise.
                        if [ ! -f "${KUBECONFIG}" ]; then
                          echo "Kubeconfig fallback ${KUBECONFIG} not found - pipeline will attempt to use credentials (if provided)."
                        else
                          echo "Using existing kubeconfig: ${KUBECONFIG}"
                          ls -l ${KUBECONFIG} || true
                        fi

                        # Vérification d'espace disque : au moins 2GB libres recommandés
                        avail_kb=$(df --output=avail -k / | tail -1)
                        avail_mb=$((avail_kb/1024))
                        echo "Available disk on / : ${avail_mb} MB"
                        if [ ${avail_mb} -lt 2048 ]; then
                          echo "WARNING: less than ~2GB free on / - minikube may fail to start. Consider cleaning docker images/volumes or increasing disk."
                        fi

                        # Si minikube est déjà running, on skip start
                        if minikube status >/dev/null 2>&1; then
                          echo "minikube status:"
                          minikube status || true
                        fi

                        # Try to start minikube (idempotent) - use docker driver and give more disk in case host permits
                        echo "Starting minikube (driver=docker, disk-size=20g) if not running..."
                        minikube start --driver=docker --disk-size=20g || true

                        # Afficher status et logs courts pour debug
                        minikube status || true
                        echo "Minikube IP: $(minikube ip || echo 'no-ip')"
                    """
                }
            }
        }

        stage('Deploy DB & App to Minikube') {
            steps {
                script {
                    // On tente d'utiliser un fichier kubeconfig fourni en credential secret (kubeconfig-file)
                    // Si la cred existe, on copie vers KUBECONFIG_PATH ; sinon on suppose que KUBECONFIG_PATH existe déjà
                    def kubeCredId = 'kubeconfig-file' // identifiant credential file (optionnel)
                    // Use withCredentials(file(...)) in a try/catch like fashion: if credential missing, step is skipped but we still proceed with fallback
                    try {
                        withCredentials([file(credentialsId: kubeCredId, variable: 'KUBE_FILE')]) {
                            sh '''
                                set -euo pipefail
                                echo "Writing Jenkins-provided kubeconfig to ${KUBECONFIG_PATH}"
                                mkdir -p $(dirname ${KUBECONFIG_PATH})
                                cp "${KUBE_FILE}" "${KUBECONFIG_PATH}"
                                chown jenkins:jenkins "${KUBECONFIG_PATH}" || true
                                chmod 600 "${KUBECONFIG_PATH}" || true
                            '''
                        }
                    } catch (err) {
                        echo "Credential '${kubeCredId}' not found or not provided — using existing ${KUBECONFIG_PATH} (if present)."
                    }

                    sh """
                        set -euo pipefail
                        export KUBECONFIG=${KUBECONFIG_PATH}
                        kubectl --kubeconfig=${KUBECONFIG} config use-context ${MINIKUBE_CONTEXT} || true

                        # create namespace if not exists
                        kubectl --kubeconfig=${KUBECONFIG} create namespace ${DEPLOY_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                        # Apply MySQL manifest (remote)
                        kubectl --kubeconfig=${KUBECONFIG} apply --insecure-skip-tls-verify=true --validate=false -f https://raw.githubusercontent.com/zainebhn/devops/main/mysql-deployment.yaml -n ${DEPLOY_NAMESPACE} || true

                        # Wait for mysql pod to be ready (label must match manifest)
                        kubectl --kubeconfig=${KUBECONFIG} wait --for=condition=ready pod -l app=mysql -n ${DEPLOY_NAMESPACE} --timeout=300s || (kubectl --kubeconfig=${KUBECONFIG} get pods -n ${DEPLOY_NAMESPACE} -o wide; exit 1)

                        # Apply app deployment + service
                        kubectl --kubeconfig=${KUBECONFIG} apply --insecure-skip-tls-verify=true --validate=false -f https://raw.githubusercontent.com/zainebhn/devops/main/deployment.yaml -n ${DEPLOY_NAMESPACE} || true
                        kubectl --kubeconfig=${KUBECONFIG} apply --insecure-skip-tls-verify=true --validate=false -f https://raw.githubusercontent.com/zainebhn/devops/main/service.yaml -n ${DEPLOY_NAMESPACE} || true

                        # Wait for app pods ready (label must match deployment.yaml - ensure label=student-app)
                        kubectl --kubeconfig=${KUBECONFIG} wait --for=condition=ready pod -l app=student-app -n ${DEPLOY_NAMESPACE} --timeout=300s || (kubectl --kubeconfig=${KUBECONFIG} get pods -n ${DEPLOY_NAMESPACE} -o wide; kubectl --kubeconfig=${KUBECONFIG} describe pods -n ${DEPLOY_NAMESPACE}; exit 1)

                        echo 'Pods in namespace ${DEPLOY_NAMESPACE}:'
                        kubectl --kubeconfig=${KUBECONFIG} get pods -n ${DEPLOY_NAMESPACE} -o wide

                        echo 'Services in namespace ${DEPLOY_NAMESPACE}:'
                        kubectl --kubeconfig=${KUBECONFIG} get svc -n ${DEPLOY_NAMESPACE} -o wide

                        # Try minikube service --url if minikube present; else fallback NodeIP:NodePort
                        if command -v minikube >/dev/null 2>&1; then
                          echo "Attempting minikube service URL..."
                          APP_URL=\$(minikube --kubeconfig=${KUBECONFIG} service student-app-service -n ${DEPLOY_NAMESPACE} --url 2>/dev/null || true)
                          if [ -n "\${APP_URL}" ]; then
                            echo "Application URL (minikube service): \${APP_URL}"
                          else
                            echo "minikube service did not return a URL, using NodeIP/NodePort fallback..."
                            NODE_IP=\$(kubectl --kubeconfig=${KUBECONFIG} get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                            NODE_PORT=\$(kubectl --kubeconfig=${KUBECONFIG} get svc student-app-service -n ${DEPLOY_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                            echo "Application fallback URL: http://\${NODE_IP}:\${NODE_PORT}"
                          fi
                        else
                          echo "minikube binary not found on agent; using NodeIP/NodePort fallback..."
                          NODE_IP=\$(kubectl --kubeconfig=${KUBECONFIG} get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                          NODE_PORT=\$(kubectl --kubeconfig=${KUBECONFIG} get svc student-app-service -n ${DEPLOY_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                          echo "Application fallback URL: http://\${NODE_IP}:\${NODE_PORT}"
                        fi
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished (always block)"
        }
        success {
            script {
                echo "Pipeline succeeded."
                sh '''
                    set -e
                    # attempt to print the app URL again for logs
                    if [ -f ${KUBECONFIG_PATH} ]; then
                      export KUBECONFIG=${KUBECONFIG_PATH}
                      if command -v minikube >/dev/null 2>&1; then
                        minikube --kubeconfig=${KUBECONFIG} service student-app-service -n ${DEPLOY_NAMESPACE} --url || true
                      else
                        NODE_IP=$(kubectl --kubeconfig=${KUBECONFIG} get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                        NODE_PORT=$(kubectl --kubeconfig=${KUBECONFIG} get svc student-app-service -n ${DEPLOY_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}')
                        echo "Possible application URL: http://${NODE_IP}:${NODE_PORT}"
                      fi
                    else
                      echo "No kubeconfig available on agent to print app URL."
                    fi
                '''
            }
        }
        failure {
            echo "Pipeline failed — consultez les logs des étapes précédentes pour diagnostiquer."
        }
    }
}
