def IMAGEN
def APP_VERSION
def jenkinsWorker = 'jenkins-worker'

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
  - name: tools
    image: 331022218908.dkr.ecr.us-east-1.amazonaws.com/tools:1.0.0 # Clients: aws oc klar 
    command:
    - cat
    tty: true
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
        NAMESPACE = "apiservice-microservicios"
        URL_OPENSHIFT = "https://api.dinersclub-dev.b6r7.p1.openshiftapps.com:6443"
    }
    options {
        timestamps ()
        timeout(time: 15, unit: 'MINUTES')
    }
    stages {
        stage('Stage: Versioning') {
            agent any
            steps {
                script {
                    // Ref: https://stackoverflow.com/a/54154911/11097939
                    IMAGEN = readMavenPom().getArtifactId()
                    // IMAGEN = readMavenPom().getArtifactId()
                    APP_VERSION = readMavenPom().getVersion()
                    echo "Nombre del Artefacto: ${IMAGEN}"
                    echo "Version: ${APP_VERSION}"

                    // sh "oc version"
                }
            }
        }
        stage('Stage: Environment') {
            agent any
            steps {
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
                    case "feature/jenkins": 
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
        stage('Stage: Build'){
            agent { 
                label "${jenkinsWorker}"
            }
            steps {
                script {
                    sh './mvnw -s settings.xml clean package -Pnative -Dmaven.test.skip=true -Dmaven.test.failure.ignore=true -Dquarkus.native.container-build=true -Dquarkus.native.container-runtime=docker -Dquarkus.container-image.build=true'
                }
            }
        }
        stage('Stage: Test'){
            agent { 
                label "${jenkinsWorker}"
            }
            stages {
                stage("Unit Test") {
                    steps {
                        script {
                            sh './mvnw -s settings.xml test'
                        }
                    }
                    
                }
                stage("Code Test") {
                    steps {
                        script {
                            withSonarQubeEnv('Sonar') {
                                echo " --> Sonar Scan"
                                sh "./mvnw -s settings.xml org.sonarsource.scanner.maven:sonar-maven-plugin:3.7.0.1746:sonar -Dsonar.projectKey=${APP_NAME}-${AMBIENTE} -Dsonar.projectName=${APP_NAME}-${AMBIENTE} -Dsonar.projectVersion=${APP_VERSION} -Dproject.settings=sonar/maven-sonar-project.properties"
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
            agent { 
                label "${jenkinsWorker}"
            }
            steps {
                script {
                    echo "Docker Build..."
                    sh "docker build -f src/main/docker/Dockerfile.native -t ${APP_NAME}-${AMBIENTE}:${APP_VERSION} ."
                    
                    echo "Docker Tag..."
                    sh "docker tag ${APP_NAME}-${AMBIENTE}:${APP_VERSION} ${PUSH}:${APP_VERSION}-${AMBIENTE}"

                    echo "Docker Push..."
                    // Credentials
                    withCredentials([usernamePassword(credentialsId: 'openshift-login', usernameVariable: 'USER_OPENSHIFT', passwordVariable: 'PASS_OPENSHIFT')]) {
                        sh label: "",
                            script: """
                                #!/bin/bash

                                set +xe
                                
                                echo " --> Login al Cluster..."
                                oc login -u \$USER_OPENSHIFT -p \$PASS_OPENSHIFT ${URL_OPENSHIFT}

                                PASS=\$( oc get secrets/aws-registry -o=go-template='{{index .data ".dockerconfigjson"}}' | base64 -d | jq -r ".[] | .[] | .password" )
                                
                                echo " --> Login al Registry..."
                                echo \$PASS | docker login --username AWS --password-stdin https://${REGISTRY}

                            """
                    }

                    sh "docker push ${PUSH}:${APP_VERSION}-${AMBIENTE}"

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
                                        
                                            PASS=\$( oc get secrets/aws-registry -o=go-template='{{index .data ".dockerconfigjson"}}' | base64 -d | jq -r ".[] | .[] | .password" )

                                            echo " --> Scanning image ${APP_NAME}-${AMBIENTE}:${APP_VERSION}..."
                                            SCAN=\$( CLAIR_ADDR=http://\$(oc get svc -l app=clair | awk '{print \$1}' | tail -1):6060 DOCKER_USER=AWS DOCKER_PASSWORD=\$PASS JSON_OUTPUT=true klar ${PUSH}:${APP_VERSION}-${AMBIENTE} )
                                            
                                            echo " --> Resultado del Scan: \$SCAN"

                                            echo " --> Validando el Scan..."
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
                                    def app = openshift.newApp("--file=./k8s/template.yaml", "--param=APP_NAME=${APP_NAME}-${AMBIENTE}", "--param=APP_VERSION=${APP_VERSION}", "--param=AMBIENTE=${AMBIENTE}", "--param=REGISTRY=${PUSH}:${APP_VERSION}-${AMBIENTE}" )
                                    
                                    // Ref: https://stackoverflow.com/a/65156451/11097939
                                    
                                    // def app = openshift.newApp("--docker-image=${PUSH}:${APP_VERSION}", "--name=${APP_NAME}-${AMBIENTE}", "--env=AMBIENTE=${AMBIENTE}", "--as-deployment-config=true", "--show-all=true").narrow('svc').expose()
                            
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
                                    openshift.set("image", "dc/${APP_NAME}-${AMBIENTE}", "${APP_NAME}-${AMBIENTE}=${PUSH}:${APP_VERSION}-${AMBIENTE}", "--record")
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
            agent { 
                label "${jenkinsWorker}"
            }
            when { 
                not { 
                    branch 'master' 
                }
            }
            steps {
                script {
                    echo " --> Cucumber Test..."
                    // sh "mvn functional-test"
                }
            }
        }
        stage('Stage: Report Functional Test') {
            agent { 
                label "${jenkinsWorker}"
            }
            when { 
                not { 
                    branch 'master' 
                }
            }
            steps {
                script {
                    echo " --> Reporte Cucumber..."
                    // cucumber '**/cucumber.json'
                    // cucumber fileIncludePattern: '**/target/cucumber.json', sortingMethod: 'ALPHABETICAL'
                }
            }
        }
        stage('Stage: Release') {
            agent { 
                label "${jenkinsWorker}"
            }
            steps {
                script {
                    echo " --> Release..."
                    def release = "v${APP_VERSION}-${env.BRANCH_NAME}"

                    // Credentials
                    withCredentials([usernamePassword(credentialsId: 'github-push', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh label: "", 
                        script: """
                            #!/bin/bash
                            
                            git config --local credential.helper "!f() { echo username=\\${GIT_USERNAME}; echo password=\\${GIT_PASSWORD}; }; f"
                            git tag ${release}

                            git push --force origin ${release}
                        
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
            node("${jenkinsWorker}") {
                // Clean Up
                script {
                    echo " ==> Cleanup..."
                    sh "docker rmi -f \$( docker images | grep none | awk '{print \$3}' ) || true"
                }
                step([$class: 'WsCleanup'])
            }
        }
    }
}

def rollback(){
    echo " --> Rollback..."
    
    REVISION = sh (script: "oc rollout history dc ${APP_NAME}-${AMBIENTE} | grep Complete | awk '{print \$1}' | tail -1 | awk '{print \$0-1}'", returnStdout:true).trim()

    echo " --> Revision: ${REVISION}"
    rollback = openshift.selector("dc/${APP_NAME}-${AMBIENTE}").rollout().undo("--to-revision=${REVISION}")
    // def result = rollback.history()
    
    def dc = openshift.selector('dc', "${APP_NAME}-${AMBIENTE}")
    // this will wait until the desired replicas are available
    dc.rollout().status()
}

