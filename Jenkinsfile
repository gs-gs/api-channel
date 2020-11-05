#!groovy

// Testing pipeline

pipeline {
    agent {
        label 'hamlet-latest'
    }
    options {
        buildDiscarder(
            logRotator(
                numToKeepStr: '10'
            )
        )
        skipDefaultCheckout()
    }

    environment {
        cd_environment = 'c1'
        slack_channel = '#igl-automatic-messages'
        properties_file = '/var/opt/properties/devnet.properties'
    }

    parameters {
        booleanParam(
            name: 'all_tests',
            defaultValue: false,
            description: 'Run tests for all components'
        )

        booleanParam(
            name: 'force_deploy',
            defaultValue: false,
            description: 'Force deployment of all components'
        )

    }

    stages {

        stage('Testing') {

            when {
                anyOf {
                    // Disable PR Testing
                    //changeRequest()
                    equals expected: true, actual: params.all_tests
                }
            }

            environment {
                COMPOSE_PROJECT_NAME="au_sg_api_channel_sg_endpoint"
            }

            stages {
                stage('Setup') {
                    steps {

                        dir("test/") {

                            script {
                                def repoSharedDb = checkout scm
                                env["GIT_COMMIT"] = repoSharedDb.GIT_COMMIT
                            }

                            echo "Starting API Channel"

                            sh '''#!/bin/bash

                                # Create external network
                                if [[ -z "$(docker network ls --filter name=igl_local_devnet --quiet)" ]]; then
                                    docker network create igl_local_devnet
                                fi

                                #Setup minio staging location


                                python pie.py -R
                                export COMPOSE_PROJECT_NAME=au_sg_api_channel_sg_endpoint

                                mkdir -p --mode=u+rwx,g+rwxs,o+rwx "${DOCKER_BUILD_DIR}/test/api-channel/docker/volumes/${COMPOSE_PROJECT_NAME}/var/minio-data/.minio.sys"
                                touch ${DOCKER_BUILD_DIR}/test/api-channel/docker/volumes/${COMPOSE_PROJECT_NAME}/var/minio-data/.minio.sys/format.json

                                python pie.py api.build
                                python pie.py api.start

                                export COMPOSE_PROJECT_NAME=au_sg_api_channel_au_endpoint

                                mkdir -p --mode=u+rwx,g+rwxs,o+rwx "${DOCKER_BUILD_DIR}/test/api-channel/docker/volumes/${COMPOSE_PROJECT_NAME}/var/minio-data/.minio.sys"
                                touch ${DOCKER_BUILD_DIR}/test/api-channel/docker/volumes/${COMPOSE_PROJECT_NAME}/var/minio-data/.minio.sys/format.json

                                python pie.py api.build
		                        python pie.py api.start

                                sleep 30s
                            '''
                        }
                    }
                }

                stage('Run Testing') {
                    steps {
                        dir('test/')  {
                            sh '''#!/bin/bash
                                export COMPOSE_PROJECT_NAME=au_sg_api_channel_au_endpoint
                                python pie.py api.test
                            '''
                        }
                    }

                    post {
                        always {
                            dir('test/') {
                                junit 'api/tests/*.xml'
                            }
                        }
                    }

                }
            }

            post {
                cleanup {
                    dir('test/') {
                        sh '''#!/bin/bash
                            python3 pie.py api.destroy
                        '''
                    }
                }
            }
        }

        stage('Artefact') {
            when {
                anyOf {
                    equals expected: true, actual: params.force_deploy
                    branch 'master'
                }

            }

            stages {
                stage('Setup') {
                    steps {
                        dir('artefact/') {
                            script {
                                def repo = checkout scm
                                env.GIT_COMMIT = repo.GIT_COMMIT
                            }
                        }

                        script {
                            def productProperties = readProperties interpolate: true, file: "${properties_file}" ;
                            productProperties.each{ k, v -> env["${k}"] ="${v}" }
                        }
                    }
                }

                // Intergov - API Lambdas
                stage('build') {
                    steps {
                        dir('artefact/serverless') {
                            sh '''#!/bin/bash
                                if [[ -d "${HOME}/.nodenv" ]]; then
                                    export PATH="$HOME/.nodenv/bin:$PATH"
                                    eval "$(nodenv init - )"
                                    nodenv install 12.16.1 || true
                                    nodenv shell 12.16.1
                                fi

                                npm ci

                                npx sls package --package dist/api_channel      --config "apis/api_channel/serverless.yml"
                            '''
                        }
                    }

                    post {
                        success {
                            dir('artefact/serverless') {
                                archiveArtifacts artifacts: 'dist/api_channel/api_channel.zip', fingerprint: true
                            }
                        }
                    }
                }

                stage('push - api_channel - lambda ') {
                    environment {
                        //hamlet deployment variables
                        DEPLOYMENT_UNITS = 'apichannel-api-imp'
                        SEGMENT = 'channel'
                        BUILD_PATH = 'artefact/'
                        BUILD_SRC_DIR = ''
                        GENERATION_CONTEXT_DEFINED = ''

                        image_format = 'lambda'
                    }

                    steps {

                        dir('artefact/serverless') {
                            sh '''
                                mkdir -p ${WORKSPACE}/artefact/dist/
                                cp dist/api_channel/channel_api.zip ${WORKSPACE}/artefact/dist/lambda.zip
                            '''
                        }

                        // Product Setup
                        sh '''#!/bin/bash
                        ${AUTOMATION_BASE_DIR}/setContext.sh || exit $?
                        '''

                        script {
                            def contextProperties = readProperties interpolate: true, file: "${WORKSPACE}/context.properties";
                            contextProperties.each{ k, v -> env["${k}"] ="${v}" }
                        }

                        sh '''#!/bin/bash
                        ${AUTOMATION_DIR}/manageImages.sh -g "${GIT_COMMIT}" -f "${image_format}"  || exit $?
                        '''

                        script {
                            def contextProperties = readProperties interpolate: true, file: "${WORKSPACE}/context.properties";
                            contextProperties.each{ k, v -> env["${k}"] ="${v}" }
                        }

                        build job: "../../deploy/deploy-${env["cd_environment"]}", wait: false, parameters: [
                                extendedChoice(name: 'DEPLOYMENT_UNITS', value: "${env.DEPLOYMENT_UNITS}"),
                                string(name: 'GIT_COMMIT', value: "${env.GIT_COMMIT}"),
                                booleanParam(name: 'AUTODEPLOY', value: true),
                                string(name: 'IMAGE_FORMATS', value: "${env.image_format}"),
                                string(name: 'SEGMENT', value: "${env.SEGMENT}")
                        ]
                    }

                }


                stage('push - api_channel - gw') {

                    environment {
                        //hamlet deployment variables
                        DEPLOYMENT_UNITS = 'apichannel-api'
                        SEGMENT = 'channel'
                        BUILD_PATH = 'artefact/api'
                        GENERATION_CONTEXT_DEFINED = ''

                        image_format = 'openapi'
                    }

                    steps {

                        dir('artefact/api') {
                                sh '''#!/bin/bash
                                # Workaround for missing api spec
                                if [[ ! -f swagger.yaml ]]; then
                                    cp ../serverless/apis/swagger.yaml ./
                                fi

                                npm install --no-save swagger-cli

                                npx swagger-cli bundle --dereference --outfile openapi-extended-base.json --type json swagger.yaml
                                npx swagger-cli validate openapi-extended-base.json

                                mkdir -p dist/
                                zip -j "dist/openapi.zip" "openapi-extended-base.json"
                            '''
                        }

                        // Product Setup
                        sh '''#!/bin/bash
                            ${AUTOMATION_BASE_DIR}/setContext.sh || exit $?
                        '''

                        script {
                            def contextProperties = readProperties interpolate: true, file: "${WORKSPACE}/context.properties";
                            contextProperties.each{ k, v -> env["${k}"] ="${v}" }
                        }

                        sh '''#!/bin/bash
                            ${AUTOMATION_DIR}/manageImages.sh -g "${GIT_COMMIT}" -f "${image_format}"  || exit $?
                        '''

                        script {
                            def contextProperties = readProperties interpolate: true, file: "${WORKSPACE}/context.properties";
                            contextProperties.each{ k, v -> env["${k}"] ="${v}" }
                        }

                        build job: "../../deploy/deploy-${env["cd_environment"]}", wait: false, parameters: [
                                extendedChoice(name: 'DEPLOYMENT_UNITS', value: "${env.DEPLOYMENT_UNITS}"),
                                string(name: 'GIT_COMMIT', value: "${env.GIT_COMMIT}"),
                                booleanParam(name: 'AUTODEPLOY', value: true),
                                string(name: 'IMAGE_FORMATS', value: "${env.image_format}"),
                                string(name: 'SEGMENT', value: "${env.SEGMENT}")
                        ]
                    }

                }

                stage('push - processors') {
                    when {
                        anyOf {
                            equals expected: true, actual: params.force_apichannel
                            allOf {
                                not {
                                    equals expected: 'master', actual: "${params.branchref_apichannel}"
                                }
                                branch 'master'
                            }
                        }
                    }

                    environment {
                        //hamlet deployment variables
                        deployment_units = 'apichannel-delivery,apichannel-spreader,apichannel-sendproc'
                        segment = 'channel'
                        image_format = 'docker'
                        BUILD_PATH = 'artefact/'
                        DOCKER_CONTEXT_DIR = 'artefact/'
                        BUILD_SRC_DIR = ''
                        DOCKER_FILE = 'artefact/docker/api.Dockerfile'
                    }

                    steps {
                        // Product Setup
                        sh '''#!/bin/bash
                            ${AUTOMATION_BASE_DIR}/setContext.sh || exit $?
                        '''

                        script {
                            def contextProperties = readProperties interpolate: true, file: "${WORKSPACE}/context.properties";
                            contextProperties.each{ k, v -> env["${k}"] ="${v}" }
                        }

                        sh '''#!/bin/bash
                            ${AUTOMATION_DIR}/manageImages.sh -g "${GIT_COMMIT}" -f "${image_format}"  || exit $?
                        '''

                        script {
                            def contextProperties = readProperties interpolate: true, file: "${WORKSPACE}/context.properties";
                            contextProperties.each{ k, v -> env["${k}"] ="${v}" }
                        }

                        build job: "../../deploy/deploy-${env["cd_environment"]}", wait: false, parameters: [
                                extendedChoice(name: 'DEPLOYMENT_UNITS', value: "${env.DEPLOYMENT_UNITS}"),
                                string(name: 'GIT_COMMIT', value: "${env.GIT_COMMIT}"),
                                booleanParam(name: 'AUTODEPLOY', value: true),
                                string(name: 'IMAGE_FORMATS', value: "${env.image_format}"),
                                string(name: 'SEGMENT', value: "${env.SEGMENT}")
                        ]
                    }

                    post {
                        success {
                            slackSend (
                                message: "Build Completed - ${BUILD_DISPLAY_NAME} (<${BUILD_URL}|Open>)\n API Channel Processer Completed",
                                channel: "#igl-automatic-messages",
                                color: "#50C878"
                            )
                        }

                        failure {
                            slackSend (
                                message: "Build Failed - ${BUILD_DISPLAY_NAME} (<${BUILD_URL}|Open>)\n API Channel Processor Failed",
                                channel: "#igl-automatic-messages",
                                color: "#B22222"
                            )
                        }
                    }
                }
            }
        }
    }
}
