def IMAGEN
def APP_VERSION

def nodeLabel = 'jenkins-job'
pipeline {
  agent {
    kubernetes {
      cloud 'openshift'
      label nodeLabel
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    identifier: ${nodeLabel}
spec:
  serviceAccountName: jenkins
  containers:
  - name: maven
    image: 331022218908.dkr.ecr.us-east-1.amazonaws.com/agent-maven:latest # quay.io/openshift/origin-jenkins-agent-maven:latest
    command:
    - cat
    tty: true
    resources:
    limits:
        cpu: 1
        memory: 1Gi
    requests:
        cpu: 0.5
        memory: 500Mi
  - name: tools
    image: 331022218908.dkr.ecr.us-east-1.amazonaws.com/tools:1.0.0 # Clients: aws oc klar 
    command:
    - cat
    tty: true
    env:
    - name: USER_OPENSHIFT
      valueFrom:
        secretKeyRef:
          key: USER_OPENSHIFT
          name: openshift-login
    - name: PASS_OPENSHIFT
      valueFrom:
        secretKeyRef:
          key: PASS_OPENSHIFT
          name: openshift-login
    - name: URL_OPENSHIFT
      valueFrom:
        secretKeyRef:
          key: URL_OPENSHIFT
          name: openshift-login
    - name: REGISTRY_OPENSHIFT
      valueFrom:
        secretKeyRef:
          key: REGISTRY_OPENSHIFT
          name: openshift-login
"""
    }
  }
    environment {
        APP_NAME = "tarjeta-credito"
        APP_VERSION = ""
        IMAGE = ""
        REGISTRY = "331022218908.dkr.ecr.us-east-1.amazonaws.com"
        REPOSITORY = "apiservice"
        PUSH = "${REGISTRY}/${REPOSITORY}"
        BASE_BUILD_IMAGE = "${REGISTRY}/openshift-s2i:latest" // https://github.com/quarkusio/quarkus-images
        NAMESPACE = "apiservice-microservicios"
    }
    options {
        timestamps ()
        timeout(time: 15, unit: 'MINUTES')
    }
    stages {
        stage('Stage: Versioning') {
            steps {
                container('tools') {
                    script {
                        // Ref: https://stackoverflow.com/a/54154911/11097939
                        IMAGEN = readMavenPom().getArtifactId()
                        // IMAGEN = readMavenPom().getArtifactId()
                        APP_VERSION = readMavenPom().getVersion()
                        echo "Nombre del Artefacto: ${IMAGEN}"
                        echo "Version: ${APP_VERSION}"

                        sh "oc version"
                    }
                }
            }
        }
        stage('Stage: Environment') {
            steps {
                container('tools') {
                    script {
                        // https://stackoverflow.com/a/59585410/11097939
                        def branch = "${env.BRANCH_NAME}"
                        echo " --> Rama: ${branch}"
                        switch(branch) {
                        case 'develop': 
                            AMBIENTE = 'dev'
                            // NAMESPACE = 'develop'
                            break
                        case "release/*": 
                            AMBIENTE = 'qa'
                            // NAMESPACE = 'qa'
                            break
                        case 'release2/*': 
                            AMBIENTE = 'uat'
                            // NAMESPACE = 'uat'
                            break
                        case 'release3/*': 
                            AMBIENTE = 'preprod'
                            // NAMESPACE = 'preproduction'
                            break  
                        case "master": 
                            AMBIENTE = 'prod' 
                            // NAMESPACE = 'production'
                            break
                        // Prueba
                        case "cicd": 
                            AMBIENTE = 'cicd'
                            break
                        default:
                            println("Branch value error: " + branch)
                            currentBuild.getRawBuild().getExecutor().interrupt(Result.FAILURE)
                        }
                        echo " --> Ambiente: ${AMBIENTE}"
                    }
                }
            }
        }
        stage('Stage: Build'){
            steps {
                container('maven') {
                    script {
                        sh 'mvn -s settings.xml clean install -Dmaven.test.skip=true -Dmaven.test.failure.ignore=true -Dquarkus.package.uber-jar=true'
                    }
                }
            }
        }
        stage('Stage: Test'){
            when { 
                not { 
                    branch 'master' 
                }
            }
            stages {
                stage("Unit Test") {
                    steps {
                        container('maven') {
                            script {
                                sh 'mvn -s settings.xml test'
                            }
                        }
                    }
                }
                stage("Code Test") {
                    steps {
                        container('maven') {
                            script {
                                withSonarQubeEnv('Sonar') {
                                    echo " --> Sonar Scan"
                                    sh "mvn -s settings.xml org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar -Dsonar.projectKey=${APP_NAME}-${AMBIENTE} -Dsonar.projectName=${APP_NAME}-${AMBIENTE} -Dsonar.projectVersion=${APP_VERSION} -Dproject.settings=sonar/maven-sonar-project.properties"
                                }
                            }
                        }
                    }
                }
                // stage('Kiuwan Test'){
                //     steps {
                //         container('kiuwan') {
                //             script {
                //                 echo " --> Kiuwan Scan"
                //                 // Ref: https://www.kiuwan.com/docs/display/K5/Jenkins+plugin
                //                 kiuwan connectionProfileUuid: 'lYfV-SD13',
                //                 sourcePath: 'folder/demo-app-repository',
                //                 applicationName: 'Demo application',
                //                 indicateLanguages: true,
                //                 languages:'java,python',
                //                 measure: 'NONE'
                //             }
                //         }
                //     }
                // }
            }
        }
        stage('Stage: Package') {
            steps {
                container('tools') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject() {
                                if (!openshift.selector("bc", "${APP_NAME}-${AMBIENTE}").exists()){
                                    echo " --> Creando el BuildConfig..."
                                    openshift.newBuild("--name=${APP_NAME}-${AMBIENTE}", "--docker-image=${BASE_BUILD_IMAGE}", "--binary", "--labels=app=${APP_NAME}-${AMBIENTE}")
                                }
                                else {
                                    echo " --> El BuildConfig existe ${APP_NAME}-${AMBIENTE}!"
                                }
                                
                                // Validando si el tag del is existe, osea la version
                                if (!openshift.selector("istag", "${APP_NAME}-${AMBIENTE}:${APP_VERSION}").exists()){
                                    echo " --> Creando la imagen para la version ${APP_VERSION}"
                                    def bc = openshift.selector("bc", "${APP_NAME}-${AMBIENTE}").startBuild("--from-file=target/${IMAGEN}-${APP_VERSION}-runner.jar", "--follow", "--wait=true")
                            
                                    def result = bc.logs('-f')
                                    echo "The logs operation require ${result.actions.size()} oc interactions"
                                    
                                    echo " --> Tagging"
                                    // Colocamos el tag latest de la imagen hecha por el build config para la version a desplegar
                                    openshift.tag("${APP_NAME}-${AMBIENTE}:latest", "${APP_NAME}-${AMBIENTE}:${APP_VERSION}")
                                }
                                else {
                                    echo " --> Esta versión existe ${APP_VERSION}-${AMBIENTE}!"
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Stage: Validate') {
            when { 
                not { 
                    branch 'master' 
                }
            }
            stages {
                stage("Container Scanner") {
                    steps {
                        container('tools') {
                            script {
                                openshift.withCluster() {
                                    openshift.withProject() {
                                        echo "Stage Clair..."                                
                                        sh label: "", 
                                        script: """
                                            #!/bin/bash

                                            set +xe
                                            
                                            # KLAR_TRACE=true

                                            echo " --> Login al Cluster..."
                                            oc login -u \$USER_OPENSHIFT -p \$PASS_OPENSHIFT \$URL_OPENSHIFT
                                            
                                            echo " --> Scanning image ${APP_NAME}-${AMBIENTE}:${APP_VERSION}..."
                                            SCAN=\$( CLAIR_ADDR=http://\$(oc get svc -l app=clair | awk '{print \$1}' | tail -1):6060 DOCKER_USER=\$(oc whoami) DOCKER_PASSWORD=\$(oc whoami -t) JSON_OUTPUT=true klar \$REGISTRY_OPENSHIFT/${NAMESPACE}/${APP_NAME}-${AMBIENTE}:${APP_VERSION} )
                                            
                                            RESULT=\$( echo \$SCAN | jq -r ".Vulnerabilities | .[] | .[] | .Severity" | grep -e Critical -e High )
                                            if [ "\$RESULT" == "" ]; then
                                                echo " --> Success! Imagen sin vulnerabilidades Critical ó High"
                                            elif [ "\$RESULT" =! "" ]; then
                                                echo " --> Error! Imagen con vulnerabilidades Critical ó High"
                                                echo " --> Scan: \$SCAN"
                                                exit 1
                                            else
                                                echo " --> Error! \$SCAN"
                                            fi

                                        """
                                        
                                        // BuildConfig para push para ECR
                                        def push = "${APP_NAME}-${AMBIENTE}-push"
                                        if (openshift.selector("bc", "${push}").exists()) {
                                            openshift.selector("bc", "${push}").delete()
                                        }
                                        echo " --> Push Image ${APP_NAME}-${AMBIENTE}:${APP_VERSION} to registry ${PUSH}..."
                                        sh label: "", 
                                        script: """
                                            #!/bin/sh
                                            
                                            set +xe
                                            
                                            echo " --> APP_NAME: ${APP_NAME}"
                                            echo " --> APP_VERSION: ${APP_VERSION}"
                                            echo " --> PUSH: ${PUSH}"

                                            oc create -f - <<-EOF
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
    labels:
    app: ${APP_NAME}-${AMBIENTE}
    name: ${APP_NAME}-${AMBIENTE}-push
spec:
    source:
        type: Binary
        binary: {}
    strategy:
        type: Source
        sourceStrategy:
            from:
                kind: ImageStreamTag
                name: "${APP_NAME}-${AMBIENTE}:${APP_VERSION}"
    output:
        to:
            kind: "DockerImage"
            name: "${PUSH}:${APP_VERSION}"
        pushSecret:
            name: aws-registry
EOF

                                        """
                                        
                                        // Push image to private registry
                                        def bc = openshift.selector("bc", "${push}").startBuild("--follow", "--wait=true")
                                        def result = bc.logs('-f')
                                        echo "The logs operation require ${result.actions.size()} oc interactions"
                                        
                                        echo " --> Se hizo push de la versión ${APP_NAME}-${AMBIENTE}:$APP_VERSION!"

                                        // Cleanup BuildConfig after push
                                        if (openshift.selector("bc", "${push}").exists()) {
                                            echo " --> Eliminando el BuildConfig del Push ${push}..."
                                            openshift.selector("bc", "${push}").delete()
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Stage: Deployment') {
            steps {
                container('tools') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject() {
                                // Validando
                                if (!openshift.selector("dc", "${APP_NAME}-${AMBIENTE}").exists()){
                                    
                                    // DeploymemtConfig
                                    echo " --> Deploy..."
                                    // Ref: https://stackoverflow.com/a/65156451/11097939
                                    
                                    def app = openshift.newApp("--docker-image=${PUSH}:${APP_VERSION}", "--name=${APP_NAME}-${AMBIENTE}", "--env=AMBIENTE=${AMBIENTE}", "--as-deployment-config=true", "--show-all=true").narrow('svc').expose()
                            
                                    def dc = openshift.selector("dc", "${APP_NAME}-${AMBIENTE}")
                                    while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                                        sleep 10
                                    }
                                    // 

                                    openshift.set("triggers", "dc/${APP_NAME}-${AMBIENTE}", "--manual")
                                    echo " --> Desployed $APP_NAME!"
                                }
                                else {
                                    echo " --> Ya existe el Deployment $APP_NAME-${AMBIENTE}!"

                                    echo " --> Updating image version..."
                                    openshift.set("image", "dc/${APP_NAME}-${AMBIENTE}", "${APP_NAME}-${AMBIENTE}=${PUSH}:${APP_VERSION}", "--record")
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Stage: Deployment Test') {
            when { 
                not { 
                    branch 'master' 
                }
            }
            steps {
                container('tools') {
                    script {
                        openshift.withCluster() {
                            openshift.withProject(){
                                // Validando el Deployment
                                // Ref: https://github.com/openshift/jenkins-client-plugin#looking-to-verify-a-deployment-or-service-we-can-still-do-that
                                echo " --> Validando el status del Deployment"
                                if (openshift.selector("dc", "${APP_NAME}-${AMBIENTE}").exists()){
                                    def latestDeploymentVersion = openshift.selector('dc',"${APP_NAME}-${AMBIENTE}").object().status.latestVersion
                                    def rc = openshift.selector('rc', "${APP_NAME}-${AMBIENTE}-${latestDeploymentVersion}")
                                    rc.untilEach(1){
                                        def rcMap = it.object()
                                        return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
                                    }
                                    
                                    def dc = openshift.selector('dc', "${APP_NAME}-${AMBIENTE}")
                                    // this will wait until the desired replicas are available
                                    def status = dc.rollout().status()
                
                                    // Validando el Service 
                                    def connected = openshift.verifyService("${APP_NAME}-${AMBIENTE}")
                                    if (connected) {
                                        echo "Able to connect to ${APP_NAME}-${AMBIENTE}"
                                    } else {
                                        echo "Unable to connect to ${APP_NAME}-${AMBIENTE}"
                                        rollback()
                                    }
                                } 
                                else {
                                    echo " --> No existe el Deployment $APP_NAME-${AMBIENTE}!"
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Stage: Functional Test') {
            when { 
                not { 
                    branch 'master' 
                }
            }
            steps {
                container('mavem') {
                    script {
                        echo " --> Cucumber Test..."
                        // sh "mvn functional-test"
                    }
                }
            }
        }
        stage('Stage: Report Functional Test') {
            when { 
                not { 
                    branch 'master' 
                }
            }
            steps {
                container('tools') {
                    script {
                        echo " --> Reporte Cucumber..."
                        // cucumber '**/cucumber.json'
                        // cucumber fileIncludePattern: '**/target/cucumber.json', sortingMethod: 'ALPHABETICAL'
                    }
                }
            }
        }
        stage('Stage: Release') {
            steps {
                container('tools') {
                    script {
                        echo " --> Release..."
                        def release = "v${APP_VERSION}-${env.BRANCH_NAME}"

                        // Credentials

                        sh label: "", 
                        script: """
                            #!/bin/bash

                            set +xe
                            
                            git config user.name 'jenkins-job'
                            git config user.email 'jenkins-job@apiservice.cl'

                            git tag ${release}
                            git push origin ${release}
                            
                            # git push https://USER_PUSH:TOKEN@https://github.com/skilledboy/tarjeta-credito.git/
                        
                        """
                    }
                }
            }
        }
        stage('Stage: Rollback') {
            steps {
                container('tools') {
                    timeout(time: 5, unit: 'MINUTES') {
                        script {
                            openshift.withCluster() {
                                openshift.withProject(){
                                    def userInputDeploy = ""

                                    userInputDeploy = input(
                                        message: '¿Ejecutar Rollback?', 
                                        ok: 'Confirmar', 
                                        parameters: [[$class: 'ChoiceParameterDefinition', 
                                        choices: 'SI\nNO\nCancelar',
                                        name: 'Seleccionar',
                                        description: 'Seleccione una opción']]
                                    )

                                    if (userInputDeploy == "SI") {
                                        // do action
                                        echo " --> Ejecutamos el rollback..."
                                        rollback()
                                    } 
                                    else if (userInputDeploy == "NO") {
                                        echo " --> No ejecutamos el rollback..."
                                    }
                                    else {
                                        // not do action
                                        echo "Action was aborted."
                                    }
                                }
                            }
                        }   
                    }
                }
            }
        }
    }
    post {
        success {
            echo " ==> SUCCES: Pipeline successful."
        }
        failure {
            echo " ==> ERROR: Pipeline failed."
        }
        always {
            // Clean Up
            script {
                echo " ==> Cleanup..."
            }
            step([$class: 'WsCleanup'])
        }
    }
}

def rollback(){
    echo " --> Rollback..."
    
    // sh label: "", 
    //     script: """
    //         #!/bin/sh
    //         REVISION=\$( oc rollout history dc ${APP_NAME}-${AMBIENTE} | grep Complete | awk '{print \$1}' | tail -1 | awk '{print \$0-1}'  )
    // """
    
    REVISION = sh (script: "oc rollout history dc ${APP_NAME}-${AMBIENTE} | grep Complete | awk '{print \$1}' | tail -1 | awk '{print \$0-1}'", returnStdout:true).trim()

    echo " --> Revision: ${REVISION}"
    rollback = openshift.selector("dc/${APP_NAME}-${AMBIENTE}").rollout().undo("--to-revision=${REVISION}")
    // def result = rollback.history()
    
    def dc = openshift.selector('dc', "${APP_NAME}-${AMBIENTE}")
    // this will wait until the desired replicas are available
    dc.rollout().status()
}
