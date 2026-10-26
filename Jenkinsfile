pipeline {
    agent { label 'podman' }
    triggers { pollSCM('H/2 * * * *') }
    environment {
        REGISTRY = 'registry.assign.nmnguyen7.300.ops:5000'
        IMAGE    = "${REGISTRY}/bind-secondary"
        TAG      = "${BUILD_NUMBER}"
    }
    stages {
        stage('Build') {
            steps { sh 'podman build -t "$IMAGE:$TAG" .' }
        }
        stage('CVE Scan') {
            steps {
                sh '''
                    mkdir -p "$HOME/.cache/trivy"
                    podman save --format docker-archive -o image.tar "$IMAGE:$TAG"
                    podman run --rm -v "$PWD:/work" -v "$HOME/.cache/trivy:/root/.cache/trivy" \
                        docker.io/aquasec/trivy:latest image --input /work/image.tar \
                        --severity HIGH,CRITICAL --exit-code 0 --no-progress \
                        --format table --output /work/trivy-report.txt
                    cat trivy-report.txt
                    rm -f image.tar
                '''
            }
            post { always { archiveArtifacts artifacts: 'trivy-report.txt', allowEmptyArchive: true } }
        }
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-login', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    sh '''
                        echo "$REG_PASS" | podman login "$REGISTRY" -u "$REG_USER" --password-stdin
                        podman push "$IMAGE:$TAG"
                    '''
                }
            }
        }
    }
    post { always { sh 'podman logout "$REGISTRY" || true' } }
}
