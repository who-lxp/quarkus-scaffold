pipeline {
    agent {
        label "master"
    }

    environment {
        // GLobal Vars
        NAME = "quarkus-scaffold"

        // Config repo managed by ArgoCD details
        ARGOCD_CONFIG_REPO = "github.com/who-lxp/lxp-config.git"
        ARGOCD_CONFIG_REPO_PATH = "lxp-deployment/values-test.yaml"
        ARGOCD_CONFIG_REPO_BRANCH = "master"

        // Job name contains the branch eg ds-app-feature%2Fjenkins-123
        JOB_NAME = "${JOB_NAME}".replace("%2F", "-").replace("/", "-")
        GIT_SSL_NO_VERIFY = true

        // Credentials bound in OpenShift
        GIT_CREDS = credentials("${OPENSHIFT_BUILD_NAMESPACE}-git-auth")
        NEXUS_CREDS = credentials("${OPENSHIFT_BUILD_NAMESPACE}-nexus-password")
        QUAY_PUSH_SECRET = "who-lxp-imagepusher-secret"

        // Nexus Artifact repos
        NEXUS_REPO_NAME="labs-static"
        NEXUS_REPO_HELM = "helm-charts"
    }

    // The options directive is for configuration that applies to the whole job.
    options {
        buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '1'))
        timeout(time: 15, unit: 'MINUTES')
        ansiColor('xterm')
    }

    stages {
        stage('Perpare Environment') {
            failFast true
            parallel {
                stage("Release Build") {
                    options {
                        skipDefaultCheckout(true)
                    }
                    agent {
                        node {
                            label "master"
                        }
                    }
                    when {
                        expression { GIT_BRANCH.startsWith("master") }
                    }
                    steps {
                        script {
                            env.TARGET_NAMESPACE = "who-lxp"
                            // External image push registry info
                            env.IMAGE_REPOSITORY = "quay.io"
                            // app name for master is just quarkus-scaffold or something
                            env.APP_NAME = "${NAME}".replace("/", "-").toLowerCase()
                        }
                    }
                }
                stage("Sandbox Build") {
                    options {
                        skipDefaultCheckout(true)
                    }
                    agent {
                        node {
                            label "master"
                        }
                    }
                    when {
                        expression { GIT_BRANCH.startsWith("dev") || GIT_BRANCH.startsWith("feature") || GIT_BRANCH.startsWith("fix") }
                    }
                    steps {
                        script {
                            env.TARGET_NAMESPACE = "my-dev"
                            // Sandbox registry deets
                            env.IMAGE_REPOSITORY = 'image-registry.openshift-image-registry.svc:5000'
                            // ammend the name to create 'sandbox' deploys based on current branch
                            env.APP_NAME = "${GIT_BRANCH}-${NAME}".replace("/", "-").toLowerCase()
                        }
                    }
                }
                stage("Pull Request Build") {
                    options {
                        skipDefaultCheckout(true)
                    }
                    agent {
                        node {
                            label "master"
                        }
                    }
                    when {
                        expression { GIT_BRANCH.startsWith("PR-") }
                    }
                    steps {
                        script {
                            env.TARGET_NAMESPACE = "my-dev"
                            env.IMAGE_REPOSITORY = 'image-registry.openshift-image-registry.svc:5000'
                            env.APP_NAME = "${GIT_BRANCH}-${NAME}".replace("/", "-").toLowerCase()
                        }
                    }
                }
            }
        }

        stage("Build (Compile App)") {
            agent {
                node {
                    label "jenkins-slave-mvn"
                }
            }
            steps {
                script {
                    env.VERSION = sh(returnStdout: true, script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout").trim()
                    env.PACKAGE = "${APP_NAME}-${VERSION}.tar.gz"
                }
                sh 'printenv'

                echo '### Java: lint, build, test, package ###'

                sh './mvnw clean deploy -s settings.xml'

                echo '### Packaging App for Nexus ###'
                sh '''
                    mv Dockerfile.jvm Dockerfile
                    tar -zcvf ${PACKAGE} Dockerfile target/lib target/*-runner.jar
                    curl -vvv -u ${NEXUS_CREDS} --upload-file ${PACKAGE} http://${SONATYPE_NEXUS_SERVICE_SERVICE_HOST}:${SONATYPE_NEXUS_SERVICE_SERVICE_PORT}/repository/${NEXUS_REPO_NAME}/${APP_NAME}/${PACKAGE}
                '''
            }
            // Post can be used both on individual stages and for the entire build.
            post {
                always {
                    // archiveArtifacts "**"
                    junit 'target/surefire-reports/*.xml'
                    // // publish html TODO
                    // publishHTML target: [
                    //     allowMissing: false,
                    //     alwaysLinkToLastBuild: false,
                    //     keepAll: true,
                    //     reportDir: 'reports/coverage/lcov-report',
                    //     reportFiles: 'index.html',
                    //     reportName: 'FE Code Coverage'
                    // ]
                }
            }
        }

        stage("Bake (OpenShift Build)") {
            options {
                skipDefaultCheckout(true)
            }
            agent {
                node {
                    label "master"
                }
            }
            steps {
                sh 'printenv'
                echo '### Get Binary from Nexus and shove it in a box ###'
                sh  '''
                    rm -rf ${PACKAGE}
                    curl -v -f -u ${NEXUS_CREDS} http://${SONATYPE_NEXUS_SERVICE_SERVICE_HOST}:${SONATYPE_NEXUS_SERVICE_SERVICE_PORT}/repository/${NEXUS_REPO_NAME}/${APP_NAME}/${PACKAGE} -o ${PACKAGE}

                    BUILD_ARGS=" --build-arg git_commit=${GIT_COMMIT} --build-arg git_url=${GIT_URL}  --build-arg build_url=${RUN_DISPLAY_URL} --build-arg build_tag=${BUILD_TAG}"
                    echo ${BUILD_ARGS}
                    oc delete bc ${APP_NAME} || rc=$?
                    if [[ $TARGET_NAMESPACE == *"dev"* ]]; then
                        echo "🏗 Creating a sandbox build for inside the cluster 🏗"
                        oc new-build --binary --name=${APP_NAME} -l app=${APP_NAME} ${BUILD_ARGS} --strategy=docker
                        oc start-build ${APP_NAME} --from-archive=${PACKAGE} ${BUILD_ARGS} --follow
                        # used for internal sandbox build ....
                        oc tag ${OPENSHIFT_BUILD_NAMESPACE}/${APP_NAME}:latest ${TARGET_NAMESPACE}/${APP_NAME}:${VERSION}
                    else
                        echo "🏗 Creating a potential build that could go all the way so pushing externally 🏗"
                        oc new-build --binary --name=${APP_NAME} -l app=${APP_NAME} ${BUILD_ARGS} --strategy=docker --push-secret=${QUAY_PUSH_SECRET} --to-docker --to="${IMAGE_REPOSITORY}/${TARGET_NAMESPACE}/${APP_NAME}:${VERSION}"
                        oc start-build ${APP_NAME} --from-archive=${PACKAGE} ${BUILD_ARGS} --follow
                    fi
                '''
            }
        }

        stage("Helm Package App") {
            agent {
                node {
                    label "jenkins-slave-helm"
                }
            }
            steps {
                sh 'printenv'
                sh '''
                    helm lint chart
                '''
                sh '''
                    # might be overkill...
                    yq w -i chart/Chart.yaml 'appVersion' ${VERSION}
                    yq w -i chart/Chart.yaml 'version' ${VERSION}

                    yq w -i chart/Chart.yaml 'name' ${APP_NAME} # APP= feature-123-quarkus-scaffold

                    # probs point to the image inside ocp cluster or perhaps an external repo?
                    yq w -i chart/values.yaml 'image_repository' ${IMAGE_REPOSITORY}
                    yq w -i chart/values.yaml 'image_name' ${APP_NAME}
                    yq w -i chart/values.yaml 'image_namespace' ${TARGET_NAMESPACE}

                    # latest built image
                    yq w -i chart/values.yaml 'app_tag' ${VERSION}
                '''
                sh '''
                    # package and release helm chart?
                    helm package chart/ --app-version ${VERSION} --version ${VERSION}
                    curl -v -f -u ${NEXUS_CREDS} http://${SONATYPE_NEXUS_SERVICE_SERVICE_HOST}:${SONATYPE_NEXUS_SERVICE_SERVICE_PORT}/repository/${NEXUS_REPO_HELM}/ --upload-file ${APP_NAME}-${VERSION}.tgz
                '''
            }
        }

        stage("Deploy App") {
            failFast true
            parallel {
                stage("sandbox - helm3 install"){
                    options {
                        skipDefaultCheckout(true)
                    }
                    agent {
                        node {
                            label "jenkins-slave-helm"
                        }
                    }
                    when {
                        expression { GIT_BRANCH.startsWith("dev") || GIT_BRANCH.startsWith("feature") || GIT_BRANCH.startsWith("fix") }
                    }
                    steps {
                        // TODO - if SANDBOX, create release in rando ns
                        sh '''
                            helm upgrade --force --install ${APP_NAME} \
                                --namespace=${TARGET_NAMESPACE} \
                                http://${SONATYPE_NEXUS_SERVICE_SERVICE_HOST}:${SONATYPE_NEXUS_SERVICE_SERVICE_PORT}/repository/${NEXUS_REPO_HELM}/${APP_NAME}-${VERSION}.tgz
                        '''
                    }
                }
                stage("test env - ArgoCD Sync") {
                    options {
                        skipDefaultCheckout(true)
                    }
                    agent {
                        node {
                            label "jenkins-slave-argocd"
                        }
                    }
                    when {
                        expression { GIT_BRANCH ==~ /(.*master)/ }
                    }
                    steps {
                        echo '### Commit new image tag to git ###'
                        sh  '''
                            git clone https://${ARGOCD_CONFIG_REPO} config-repo
                            cd config-repo
                            git checkout ${ARGOCD_CONFIG_REPO_BRANCH}

                            yq w -i ${ARGOCD_CONFIG_REPO_PATH} "applications.name==test-${NAME}.source_ref" ${VERSION}

                            git config --global user.email "jenkins@rht-labs.bot.com"
                            git config --global user.name "Jenkins"
                            git config --global push.default simple

                            git add ${ARGOCD_CONFIG_REPO_PATH}
                            git commit -m "🚀 AUTOMATED COMMIT - Deployment new app version ${VERSION} 🚀" || rc=$?
                            git remote set-url origin  https://${GIT_CREDS_USR}:${GIT_CREDS_PSW}@${ARGOCD_CONFIG_REPO}
                            git push -u origin ${ARGOCD_CONFIG_REPO_BRANCH}
                        '''
                    }
                }
            }
        }

        stage("Trigger System Tests") {
            options {
                skipDefaultCheckout(true)
            }
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression { GIT_BRANCH ==~ /(.*master)/ }
            }
            steps {
                sh  '''
                    echo "TODO - Run tests"
                '''
                build job: 'system-tests/master', parameters: [[$class: 'StringParameterValue', name: 'APP_NAME', value: "${APP_NAME}" ],[$class: 'StringParameterValue', name: 'VERSION', value: "${VERSION}"]], wait: false
            }
        }
    }
}
