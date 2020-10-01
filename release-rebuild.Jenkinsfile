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
          indy-pipeline-build-number: "${env.BUILD_NUMBER}"
      spec:
        containers:
        - name: jnlp
          image: quay.io/kaine/indy-stress-tester:latest
          imagePullPolicy: Always
          tty: true
          env:
          - name: USER
            value: 'jenkins-k8s-config'
          - name: HOME
            value: /home/jenkins
          - name: JAVA_TOOL_OPTIONS
            value: '-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Dsun.zip.disableMemoryMapping=true -Xms1024m -Xmx5g'
          - name: MAVEN_OPTS
            value: -Xmx8g -Xms1024m -XX:MaxPermSize=512m -Xss8m
          - name: NPMREGISTRY
            value: 'https://repository.engineering.redhat.com/nexus/repository/registry.npmjs.org'
          resources:
            requests:
              memory: 6Gi
              cpu: 2000m
            limits:
              memory: 6Gi
              cpu: 2000m
          volumeMounts:
          - mountPath: /home/jenkins/sonatype
            name: volume-0
          - mountPath: /home/jenkins/gnupg_keys
            name: volume-1
          - mountPath: /mnt/ocp
            name: volume-2
          workingDir: /home/jenkins
        volumes:
        - name: volume-0
          secret:
            defaultMode: 420
            secretName: sonatype-secrets
        - name: volume-1
          secret:
            defaultMode: 420
            secretName: gnupg
        - name: volume-2
          configMap:
            defaultMode: 420
            name: jenkins-openshift-mappings
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
