pipeline {
  agent {
    kubernetes {
      cloud params.JENKINS_AGENT_CLOUD_NAME
      label "jenkins-slave-${UUID.randomUUID().toString()}"
      serviceAccount "jenkins"
      defaultContainer 'jnlp'
      yaml """
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: "jenkins-${env.JOB_BASE_NAME}"
      spec:
        containers:
        - name: jnlp
          image: registry.redhat.io/openshift3/jenkins-agent-maven-35-rhel7:v3.11.219-1
          imagePullPolicy: Always
          tty: true
          env:
          - name: USER
            value: 'jenkins-k8s-config'
          - name: HOME
            value: /home/jenkins
          resources:
            requests:
              memory: 4Gi
              cpu: 2000m
            limits:
              memory: 5Gi
              cpu: 4000m
          workingDir: /home/jenkins
      """
    }
  }
  options {
    //timestamps()
    timeout(time: 180, unit: 'MINUTES')
  }
  environment {
    PIPELINE_NAMESPACE = readFile('/run/secrets/kubernetes.io/serviceaccount/namespace').trim()
    PIPELINE_USERNAME = sh(returnStdout: true, script: 'id -un').trim()
  }
  stages {
    stage('respin release image'){
      steps{
        script{
          openshift.withCluster(){
            def template = readYaml file: 'openshift/indy-quay-template.yaml'
            def processed = openshift.process(template,
              '-p', "TARBALL_URL=https://github.com/Commonjava/indy/releases/download/indy-parent-${params.INDY_VERSION}/indy-launcher-${params.INDY_VERSION}-skinny.tar.gz",
              '-p', "DATA_TARBALL_URL=https://github.com/Commonjava/indy/releases/download/indy-parent-${params.INDY_VERSION}/indy-launcher-${params.INDY_VERSION}-data.tar.gz",
              '-p', "INDY_VERSION=${params.INDY_VERSION}",
              '-p', "QUAY_TAG=v${params.INDY_VERSION}"
            )
            def build = c3i.buildAndWait(script: this, objs: processed)
            echo 'Image respin successful!'
          }
        }
      }
    }
  }
}
