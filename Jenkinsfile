pipeline {
  agent any
  environment {
    DOCKER_REGISTRY = credentials('docker-registry-credentials') // username/password pair ID or use DOCKER_CREDS
    IMAGE_NAME = "sweetyraj22/e-learning-site"          // replace
                                         // optional
  }
  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Image') {
      steps {
        script {
          // use git commit hash as tag
          GIT_COMMIT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          IMAGE_TAG = "${env.IMAGE_NAME}:${GIT_COMMIT}"
        }
        sh "docker build -t ${IMAGE_TAG} ."
      }
    }

    stage('Push Image') {
      steps {
        script {
          // login then push - expect DOCKER_REGISTRY to be username/password credentials id in Jenkins
          withCredentials([usernamePassword(credentialsId: 'docker-registry-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
            sh "docker push ${IMAGE_TAG}"
          }
        }
      }
    }

    stage('Deploy to Kubernetes (blue/green)') {
      steps {
        script {
          // Determine which color is idle (flip)
          // Requires kubectl configured on the agent with access to cluster
          // We'll detect which version service currently points to and deploy to the other
          def svcSelector = sh(script: "kubectl get svc web-svc -o jsonpath='{.spec.selector.version}' || echo ''", returnStdout: true).trim()
          echo "Current service selector version: ${svcSelector}"
          def newColor = (svcSelector == 'blue' || svcSelector == '') ? 'green' : 'blue'
          echo "Will deploy to: ${newColor}"

          // Apply the correct deployment manifest with IMAGE replacement
          def deployFile = "k8s/deployment-${newColor}.yaml"
          sh "sed 's|REPLACE_IMAGE|${IMAGE_TAG}|g' ${deployFile} > /tmp/deploy-${newColor}.yaml"
          sh "kubectl apply -f /tmp/deploy-${newColor}.yaml"

          // Wait for rollout to finish
          sh "kubectl rollout status deployment/web-${newColor} --timeout=120s"

          // Smoke test the new deployment by port-forwarding or by switching service temporarily (we will try a readiness probe check)
          // Option A: test pods are Ready
          sh """
            POD=\$(kubectl get pods -l app=web,version=${newColor} -o jsonpath='{.items[0].metadata.name}')
            echo "Testing pod: \$POD"
            kubectl exec \$POD -- wget -qO- --tries=3 --timeout=5 http://localhost/ || (echo 'smoke test failed' && exit 1)
          """
          // Now switch service selector to the new color
          sh "kubectl -n default patch service web-svc -p \"{\\\"spec\\\":{\\\"selector\\\":{\\\"app\\\":\\\"web\\\",\\\"version\\\":\\\"${newColor}\\\"}}}\""

          // Wait a few seconds for service to route
          sleep 5

          // Final smoke test by curling the service (assuming cluster DNS accessible from agent)
          sh "kubectl run curl-test --image=appropriate/curl --rm -i --restart=Never --command -- curl -fsS http://web-svc.default.svc.cluster.local/ || (echo 'service test failed' && exit 1)"

          // Scale down the old deployment
          def oldColor = (newColor == 'blue') ? 'green' : 'blue'
          echo "Scaling down old deployment: ${oldColor}"
          sh "kubectl scale deployment web-${oldColor} --replicas=0 || true"
        }
      }
    }
  }
  post {
    success {
      echo "Blue-Green deployment successful. Image: ${IMAGE_TAG}"
    }
    failure {
      echo "Deployment failed — keep previous serving version. Manual investigation required."
    }
  }
}


