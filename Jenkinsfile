pipeline {
    agent {
        label "maven"
    }
    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }
    stages {
        stage("Checkout") {
            steps {     
                library(identifier: "openshift-pipeline-library@master", 
                        retriever: modernSCM([$class: "GitSCMSource",
                                              credentialsId: "dev-repository-credentials",
                                              remote: "https://github.com/jaysonzhao/openshift-cicd-demo.git"]))                
                
                initParameters() 
                
                gitClone()

                stash "repo"
            }
        }
        stage("Compile") {
            steps {
                sh "mvn package -DskipTests"
            }
        }
        stage("Test") {
            steps {
                sh "mvn test"
            }
        }
        stage("Build Image") {
            steps {
                applyTemplate(project: env.DEV_PROJECT, 
                              application: env.APP_NAME, 
                              template: env.APP_TEMPLATE, 
                              parameters: env.APP_TEMPLATE_PARAMETERS_DEV,
                              createBuildObjects: true)

                buildImage(project: env.DEV_PROJECT, 
                           application: env.APP_NAME, 
                           artifactsDir: "./target")
            }
        }
        stage("Deploy DEV") {
            steps {
                script {
                    env.TAG_NAME = readMavenPom().getVersion()
                }   
                
                tagImage(srcProject: env.DEV_PROJECT, 
                         srcImage: env.IMAGE_NAME, 
                         srcTag: "latest", 
                         dstProject: env.DEV_PROJECT, 
                         dstImage: env.IMAGE_NAME,
                         dstTag: env.TAG_NAME)
                
                deployImage(project: env.DEV_PROJECT, 
                            application: env.APP_NAME, 
                            image: env.IMAGE_NAME, 
                            tag: env.TAG_NAME)
            }
        }
        stage("Deploy TEST") {
            steps {
               // input("Promote to TEST?")

                applyTemplate(project: env.TEST_PROJECT, 
                              application: env.APP_NAME, 
                              template: env.APP_TEMPLATE, 
                              parameters: env.APP_TEMPLATE_PARAMETERS_TEST)

                tagImage(srcProject: env.DEV_PROJECT, 
                         srcImage: env.IMAGE_NAME, 
                         srcTag: env.TAG_NAME, 
                         dstProject: env.TEST_PROJECT, 
                         dstImage: env.IMAGE_NAME,
                         dstTag: env.TAG_NAME)
                
                deployImage(project: env.TEST_PROJECT, 
                            application: env.APP_NAME, 
                            image: env.IMAGE_NAME, 
                            tag: env.TAG_NAME)
            }
        }
        stage("Integration Test") {
            agent {
                kubernetes {
                    cloud "openshift"
                    defaultContainer "jnlp"
                    label "${env.APP_NAME}-int-test"
                    yaml """
                        apiVersion: v1
                        kind: Pod
                        spec:
                          containers:
                          - name: python
                            image:  docker-registry.default.svc:5000/openshift/python:3.6                        
                            command:
                            - cat
                            tty: true
                    """                
                }
            }
            steps {
                unstash "repo"

                container("python") {
                    sh "pip install requests"
                    sh "python ./src/test/python/it.py"
                }
            }
        }
        stage("Deploy PROD") {
            steps {
                script {
                   
                  input("Promote to PROD?")  
                   
                } 
            }
        }
        
    }
}
