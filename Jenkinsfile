G_giturl = ""
G_gitcred = 'TonJenSSH'
G_docker_creds = "TonJenDockerHub"
G_params = null
G_images = [:]
G_docker_image = null
G_commit = ""
G_binversion = "NotSet"

def isUpstream() {
    return currentBuild.getBuildCauses()[0]._class.toString() == 'hudson.model.Cause$UpstreamCause'
}

def getVar(Gvar) {
    return Gvar
}

pipeline {
    agent {
        node 'master'
    }
    tools {nodejs "Node12.8.0"}
    options {
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '20')
        
        parallelsAlwaysFailFast()
    }
    parameters {
        string(
            name:'common_version',
            defaultValue: '',
            description: 'Common version'
        )
        string(
            name:'image_ton_labs_types',
            defaultValue: '',
            description: 'ton-labs-types image name'
        )
        string(
            name:'image_ton_labs_block',
            defaultValue: '',
            description: 'ton-labs-block image name'
        )
        string(
            name:'image_ton_labs_vm',
            defaultValue: '',
            description: 'ton-labs-vm image name'
        )
        string(
            name:'image_ton_labs_abi',
            defaultValue: '',
            description: 'ton-labs-abi image name'
        )
        string(
            name:'image_ton_executor',
            defaultValue: '',
            description: 'ton-executor image name'
        )
        string(
            name:'image_ton_sdk',
            defaultValue: '',
            description: 'ton-sdk image name'
        )
    }
    stages {
        stage('Initialize') {
            steps {
                script {
                    G_giturl = env.GIT_URL
                    echo "${G_giturl}"
                    if(isUpstream() && GIT_BRANCH != "master") {
                        checkout(
                            [$class: 'GitSCM', 
                            branches: [[name: "origin/${GIT_BRANCH}"]], 
                            doGenerateSubmoduleConfigurations: false, 
                            extensions: [[
                                $class: 'PreBuildMerge', 
                                options: [
                                    mergeRemote: 'origin', 
                                    mergeTarget: 'master'
                                ]
                            ]], 
                            submoduleCfg: [], 
                            userRemoteConfigs: [[credentialsId: 'TonJen', url: G_giturl]]])
                        G_commit = sh (script: 'git rev-parse HEAD^{commit}', returnStdout: true).trim()
                        echo "${GIT_COMMIT} merged with master. New commit ${G_commit}"
                    } else {
                        G_commit = GIT_COMMIT
                    }
                    C_TEXT = sh (script: "git show -s --format=%s ${GIT_COMMIT}", \
                        returnStdout: true).trim()
                    C_AUTHOR = sh (script: "git show -s --format=%an ${GIT_COMMIT}", \
                        returnStdout: true).trim()
                    C_COMMITER = sh (script: "git show -s --format=%cn ${GIT_COMMIT}", \
                        returnStdout: true).trim()
                    C_HASH = sh (script: "git show -s --format=%h ${GIT_COMMIT}", \
                        returnStdout: true).trim()
                    C_PROJECT = G_giturl.substring(15,G_giturl.length()-4)
                    C_GITURL = sh (script: "echo ${GIT_URL}",returnStdout: true).trim()
                    C_GITCOMMIT = sh (script: "echo ${GIT_COMMIT}", \
                        returnStdout: true).trim()
                    if(params.image_ton_labs_types) {
                        G_images.put('ton-labs-types', params.image_ton_labs_types)
                    } else {
                        G_images.put('ton-labs-types', "tonlabs/ton-labs-types:latest")
                    }
                    if(params.image_ton_labs_block) {
                        G_images.put('ton-labs-block', params.image_ton_labs_block)
                    } else {
                        G_images.put('ton-labs-block', "tonlabs/ton-labs-block:latest")
                    }
                    if(params.image_ton_labs_vm) {
                        G_images.put('ton-labs-vm', params.image_ton_labs_vm)
                    } else {
                        G_images.put('ton-labs-vm', "tonlabs/ton-labs-vm:latest")
                    }
                    if(params.image_ton_labs_abi) {
                        G_images.put('ton-labs-abi', params.image_ton_labs_abi)
                    } else {
                        G_images.put('ton-labs-abi', "tonlabs/ton-labs-abi:latest")
                    }
                    if(params.image_ton_executor) {
                        G_images.put('ton-executor', params.image_ton_executor)
                    } else {
                        G_images.put('ton-executor', "tonlabs/ton-executor:latest")
                    }
                    if(params.image_ton_sdk) {
                        G_images.put('TON-SDK', params.image_ton_sdk)
                    } else {
                        G_images.put('TON-SDK', "tonlabs/ton-sdk:source-${G_commit}")
                    }
                    env.IMAGE = G_images['TON-SDK']                        
                }
                echo "Version: ${getVar(G_binversion)}."
                echo "Branch: ${GIT_BRANCH}"
                echo "Possible RC: ${getVar(G_binversion)}-rc"
            }
        }
        stage('Versioning') {
            steps {
                script {
                    lock('bucket') {
                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                            identity = awsIdentity()
                            s3Download bucket: 'sdkbinaries.tonlabs.io', file: 'version.json', force: true, path: 'version.json'
                        }
                    }
                    def folders = """ton_sdk \
ton_client/client \
ton_client/platforms/ton-client-node-js \
ton_client/platforms/ton-client-react-native \
ton_client/platforms/ton-client-web"""
                    if(params.common_version) {
                        G_binversion = sh (script: "node tonVersion.js --set ${params.common_version} ${folders}", returnStdout: true).trim()
                    } else {
                        G_binversion = sh (script: "node tonVersion.js ${folders}", returnStdout: true).trim()
                    }
                }
            }
        }
        stage('Switch to file source') {
            steps {
                sh """
                    node pathFix.js ton_sdk/Cargo.toml \"ton_abi = {.*\" \"ton_abi = { path = \\\"/tonlabs/ton-labs-abi\\\" }\"
                    node pathFix.js ton_sdk/Cargo.toml \"ton_block = {.*\" \"ton_block = { path = \\\"/tonlabs/ton-labs-block\\\" }\"
                    node pathFix.js ton_sdk/Cargo.toml \"ton_vm = {.*\" \"ton_vm = { path = \\\"/tonlabs/ton-labs-vm\\\", default-features = false }\"
                    node pathFix.js ton_sdk/Cargo.toml \"ton_types = {.*\" \"ton_types = { path = \\\"/tonlabs/ton-labs-types\\\" }\"
                    node pathFix.js ton_sdk/Cargo.toml \"ton_executor = {.*\" \"ton_executor = { path = \\\"/tonlabs/ton-executor\\\" }\"
                    node pathFix.js ton_sdk/Cargo.toml \"git = \\\"ssh://git@github.com/tonlabs/ton-executor.git\\\"\" \"path = \\\"/tonlabs/ton-executor\\\"\"
                    node pathFix.js ton_client/client/Cargo.toml \"ton_block = {.*\" \"ton_block = { path = \\\"/tonlabs/ton-labs-block\\\" }\"
                    node pathFix.js ton_client/client/Cargo.toml \"ton_vm = {.*\" \"ton_vm = { path = \\\"/tonlabs/ton-labs-vm\\\", default-features = false }\"
                    node pathFix.js ton_client/client/Cargo.toml \"ton_types = {.*\" \"ton_types = { path = \\\"/tonlabs/ton-labs-types\\\" }\"
                """
            }
        }
        stage('Build sources image') {
            steps {
                script {
                    docker.withRegistry('', G_docker_creds) {
                        sshagent (credentials: [G_gitcred]) {
                            withEnv(["DOCKER_BUILDKIT=1", "BUILD_INFO=${env.BUILD_TAG}:${GIT_COMMIT}"]) {
                                src_image = docker.build(
                                    "${G_images['ton-sdk']}",
                                    "--pull --label \"git-commit=\${GIT_COMMIT}\" --target ton-sdk-src ."
                                )
                            }
                        }
                        docker.image("${G_images['ton-sdk']}").push()
                    }
                }
            }
        }
        stage('Build common sources for agents') {
            agent {
                dockerfile {
                    registryCredentialsId "${G_docker_creds}"
                    additionalBuildArgs "--pull --target ton-sdk-full " + 
                                        "--build-arg \"TON_LABS_TYPES_IMAGE=${G_images['ton-labs-types']}\" " +
                                        "--build-arg \"TON_LABS_BLOCK_IMAGE=${G_images['ton-labs-block']}\" " + 
                                        "--build-arg \"TON_LABS_VM_IMAGE=${G_images['ton-labs-vm']}\" " + 
                                        "--build-arg \"TON_LABS_ABI_IMAGE=${G_images['ton-labs-abi']}\" " + 
                                        "--build-arg \"TON_EXECUTOR_IMAGE=${G_images['ton-executor']}\" " +
                                        "--build-arg \"TON_SDK_IMAGE=${G_images['ton-sdk']}\""
                }
            }
            steps {
                script {
                    sh """
                        zip -9 -r ton-sdk-src.zip /tonlabs/*
                        ls -la ton-sdk-src.zip
                        chown jenkins:jenkins ton-sdk-src.zip
                    """
                    stash includes: '**/ton-sdk-src.zip', name: 'ton-sdk-src'
                }
            }
        }
        stage('Building...') {
            failFast true
            parallel {
                stage('Client linux') {
                    agent {
                        dockerfile {
                            registryCredentialsId "${G_docker_creds}"
                            additionalBuildArgs "--pull --target ton-sdk-rust " + 
                                                "--build-arg \"TON_LABS_TYPES_IMAGE=${G_images['ton-labs-types']}\" " +
                                                "--build-arg \"TON_LABS_BLOCK_IMAGE=${G_images['ton-labs-block']}\" " + 
                                                "--build-arg \"TON_LABS_VM_IMAGE=${G_images['ton-labs-vm']}\" " + 
                                                "--build-arg \"TON_LABS_ABI_IMAGE=${G_images['ton-labs-abi']}\" " + 
                                                "--build-arg \"TON_EXECUTOR_IMAGE=${G_images['ton-executor']}\" " +
                                                "--build-arg \"TON_SDK_IMAGE=${G_images['ton-sdk']}\""
                        }
                    }
                    stages {
                        stage('Build') {
                            steps {
                                script {
                                    sh """
                                        cd /tonlabs/TON-SDK/ton_client/client
                                        node build.js
                                    """
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                script {
                                    sh "cp /tonlabs/TON-SDK/ton_client/client/bin/*gz ./"
                                    stash allowEmpty: true, includes: '*.gz', name: 'cl-linux-bin'
                                    /*
                                    withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                        identity = awsIdentity()
                                        s3Upload \
                                            bucket: 'sdkbinaries.tonlabs.io', \
                                            includePathPattern:'*.gz', path: 'tmp_sdk', \
                                            workingDir:'.'
                                    }
                                    */
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}
				}
                stage('Client macOS') {
                    agent {
                        label "ios"
                    }
                    stages {
                        stage('Report versions') {
                            steps {
                                sh 'rustc --version'
                                sh 'cargo --version'
                            }
                        }
                        stage('Get sources') {
                            steps {
                                script {
                                    def C_PATH = sh (script: 'pwd', returnStdout: true).trim()
                            
                                    unstash 'ton-sdk-src'
                                    sh """
                                        rm Cargo.toml
                                        unzip ton-sdk-src.zip
                                        node pathFix.js tonlabs/ton-labs-block/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-vm/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-abi/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-executor/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_sdk/Cargo.toml \"path = \\\"/tonlabs/\" \"path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/client/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-web/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                    """
                                }
                            }
                        }
                        stage('Build') {
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/client') {
                                    sshagent([G_gitcred]) {
                                        sh 'node build.js'
                                    }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/client/bin') {
                                    script {
                                        stash allowEmpty: true, includes: '*.gz', name: 'cl-darwin-bin'
                                        /*
                                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                            identity = awsIdentity()
                                            s3Upload \
                                                bucket: 'sdkbinaries.tonlabs.io', \
                                                includePathPattern:'*.gz', path: 'tmp_sdk', \
                                                workingDir:'.'
                                        }
                                        */
                                    }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}
				}
                stage('Client Windows') {
                    agent {
                        label "Win"
                    }
                    stages {
                        stage('Report versions') {
                            steps {
                                bat 'rustc --version'
                                bat 'cargo --version'
                            }
                        }
                        stage('Get sources') {
                            steps {
                                script {
                                    def C_PATH = bat (script: '@echo off && echo %cd%', returnStdout: true).trim()
                            
                                    unstash 'ton-sdk-src'
                                    bat """
                                        del Cargo.toml
                                        unzip ton-sdk-src.zip
                                        node pathFix.js tonlabs\\ton-labs-block\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\ton-labs-vm\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\ton-labs-abi\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\ton-executor\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_sdk\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_sdk\\Cargo.toml \"/tonlabs/ton-executor\" \"${C_PATH}\\tonlabs\\ton-executor\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\client\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\platforms\\ton-client-node-js\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\platforms\\ton-client-react-native\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\platforms\\ton-client-web\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                    """
                                }
                            }
                        }
                        stage('Build') {
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/client') {
                                    sshagent([G_gitcred]) {
                                        bat 'node build.js'
                                    }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/client/bin') {
                                    script {
                                        stash allowEmpty: true, includes: '*.gz', name: 'cl-windows-bin'
                                        /*
                                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                            identity = awsIdentity()
                                            s3Upload \
                                                bucket: 'sdkbinaries.tonlabs.io', \
                                                includePathPattern:'*.gz', path: 'tmp_sdk', \
                                                workingDir:'.'
                                        }
                                        */
                                    }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}                
				}
                stage('react-native-ios') {
                    agent {
                        label "ios"
                    }
                    stages {
                        stage('Report versions') {
                            steps {
                                sh '''
                                rustc --version
                                cargo --version
                                '''
                            }
                        }
                        stage('Get sources') {
                            steps {
                                script {
                                    def C_PATH = sh (script: 'pwd', returnStdout: true).trim()
                            
                                    unstash 'ton-sdk-src'
                                    sh """
                                        rm Cargo.toml
                                        unzip ton-sdk-src.zip
                                        node pathFix.js tonlabs/ton-labs-block/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-vm/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-abi/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-executor/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_sdk/Cargo.toml \"path = \\\"/tonlabs/\" \"path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/client/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-web/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                    """
                                }
                            }
                        }
                        stage('Build') {
                            steps {
                                echo 'Build ...'
                                sshagent([G_gitcred]) {
                                    dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native') {
                                        sh 'node build.js --ios'
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_build = false }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native/output') {
                                    script {
                                        stash allowEmpty: true, includes: '*.gz', name: 'rn-ios-bin'
                                        /*
                                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                            identity = awsIdentity()
                                            s3Upload \
                                                bucket: 'sdkbinaries.tonlabs.io', \
                                                includePathPattern:'*.gz', path: 'tmp_sdk', \
                                                workingDir:'.'
                                        }
                                        */
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_deploy = false }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}                
				}
                stage('react-native-android') {
                    agent {
                        label "ios"
                    }
                    stages {
                        stage('Report versions') {
                            steps {
                                sh '''
                                rustc --version
                                cargo --version
                                '''
                            }
                        }
                        stage('Get sources') {
                            steps {
                                script {
                                    def C_PATH = sh (script: 'pwd', returnStdout: true).trim()
                            
                                    unstash 'ton-sdk-src'
                                    sh """
                                        rm Cargo.toml
                                        unzip ton-sdk-src.zip
                                        node pathFix.js tonlabs/ton-labs-block/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-vm/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-abi/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-executor/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_sdk/Cargo.toml \"path = \\\"/tonlabs/\" \"path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/client/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-web/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                    """
                                }
                            }
                        }
                        stage('Build') {
                            steps {
                                echo 'Build ...'
                                sshagent([G_gitcred]) {
                                    dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native') {
                                        sh 'node build.js --android'
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_build = false }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native/output') {
                                    script {
                                        stash allowEmpty: true, includes: '*.gz', name: 'rn-android-bin'
                                        /*
                                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                            identity = awsIdentity()
                                            s3Upload \
                                                bucket: 'sdkbinaries.tonlabs.io', \
                                                includePathPattern:'*.gz', path: 'tmp_sdk', \
                                                workingDir:'.'
                                        }
                                        */
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_deploy = false }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}                
				}
                stage('node-js for iOS') {
                    agent {
                        label "ios"
                    }
                    stages {
                        stage('Report versions') {
                            steps {
                                sh '''
                                rustc --version
                                cargo --version
                                '''
                            }
                        }
                        stage('Get sources') {
                            steps {
                                script {
                                    def C_PATH = sh (script: 'pwd', returnStdout: true).trim()
                            
                                    unstash 'ton-sdk-src'
                                    sh """
                                        rm Cargo.toml
                                        unzip ton-sdk-src.zip
                                        node pathFix.js tonlabs/ton-labs-block/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-vm/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-labs-abi/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/ton-executor/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_sdk/Cargo.toml \"path = \\\"/tonlabs/\" \"path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/client/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-react-native/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                        node pathFix.js tonlabs/TON-SDK/ton_client/platforms/ton-client-web/Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}/tonlabs/\"
                                    """
                                }
                            }
                        }
                        stage('Build') {
                            steps {
                                echo 'Build ...'
                                sshagent([G_gitcred]) {
                                    dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js') {
                                        sh 'node build.js'
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_build = false }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js/bin') {
                                    script {
                                        stash allowEmpty: true, includes: '*.gz', name: 'nj-darwin-bin'
                                        /*
                                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                            identity = awsIdentity()
                                            s3Upload \
                                                bucket: 'sdkbinaries.tonlabs.io', \
                                                includePathPattern:'*.gz', path: 'tmp_sdk', \
                                                workingDir:'.'
                                        }
                                        */
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_deploy = false }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}                
				}
                stage('node-js for Windows') {
                    agent {
                        label "Win"
                    }
                    stages {
                        stage('Report versions') {
                            steps {
                                bat '''
                                rustc --version
                                cargo --version
                                '''
                            }
                        }
                        stage('Get sources') {
                            steps {
                                script {
                                    def C_PATH = bat (script: '@echo off && echo %cd%', returnStdout: true).trim()
                            
                                    unstash 'ton-sdk-src'
                                    bat """
                                        del Cargo.toml
                                        unzip ton-sdk-src.zip

                                        node pathFix.js tonlabs\\ton-labs-block\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\ton-labs-vm\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\ton-labs-abi\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\ton-executor\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_sdk\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_sdk\\Cargo.toml \"/tonlabs/ton-executor\" \"${C_PATH}\\tonlabs\\ton-executor\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\client\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\platforms\\ton-client-node-js\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\platforms\\ton-client-react-native\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                        node pathFix.js tonlabs\\TON-SDK\\ton_client\\platforms\\ton-client-web\\Cargo.toml \"{ path = \\\"/tonlabs/\" \"{ path = \\\"${C_PATH}\\tonlabs\\\\\"
                                    """
                                }
                            }
                        }
                        stage('Build') {
                            steps {
                                echo 'Build ...'
                                sshagent([G_gitcred]) {
                                    dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js') {
                                        bat 'node build.js'
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_build = false }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                dir('tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js/bin') {
                                    script {
                                        stash allowEmpty: true, includes: '*.gz', name: 'nj-windows-bin'
                                        /*
                                        withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                            identity = awsIdentity()
                                            s3Upload \
                                                bucket: 'sdkbinaries.tonlabs.io', \
                                                includePathPattern:'*.gz', path: 'tmp_sdk', \
                                                workingDir:'.'
                                        }
                                        */
                                    }
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_deploy = false }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}                
				}
                stage('node-js for Linux') {
                    agent {
                        dockerfile {
                            registryCredentialsId "${G_docker_creds}"
                            additionalBuildArgs "--pull --target ton-sdk-rust " + 
                                                "--build-arg \"TON_LABS_TYPES_IMAGE=${G_images['ton-labs-types']}\" " +
                                                "--build-arg \"TON_LABS_BLOCK_IMAGE=${G_images['ton-labs-block']}\" " + 
                                                "--build-arg \"TON_LABS_VM_IMAGE=${G_images['ton-labs-vm']}\" " + 
                                                "--build-arg \"TON_LABS_ABI_IMAGE=${G_images['ton-labs-abi']}\" " + 
                                                "--build-arg \"TON_EXECUTOR_IMAGE=${G_images['ton-executor']}\" " +
                                                "--build-arg \"TON_SDK_IMAGE=${G_images['ton-sdk']}\""
                        }
                    }
                    stages {
                        stage('Build') {
                            steps {
                                sshagent([G_gitcred]) {
                                    sh """
                                        cd /tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js
                                        node build.js
                                    """
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_build = false }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                script {
                                    sh "cp /tonlabs/TON-SDK/ton_client/platforms/ton-client-node-js/bin/*.gz ./"
                                    stash allowEmpty: true, includes: '*.gz', name: 'nj-linux-bin'
                                    /*
                                    withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                        identity = awsIdentity()
                                        s3Upload \
                                            bucket: 'sdkbinaries.tonlabs.io', \
                                            includePathPattern:'*.gz', path: 'tmp_sdk', \
                                            workingDir:'.'
                                    }
                                    */
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_deploy = false }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}                
				}
                stage('web') {
                    agent {
                        dockerfile {
                            registryCredentialsId "${G_docker_creds}"
                            additionalBuildArgs "--pull --target ton-sdk-rust " + 
                                                "--build-arg \"TON_LABS_TYPES_IMAGE=${G_images['ton-labs-types']}\" " +
                                                "--build-arg \"TON_LABS_BLOCK_IMAGE=${G_images['ton-labs-block']}\" " + 
                                                "--build-arg \"TON_LABS_VM_IMAGE=${G_images['ton-labs-vm']}\" " + 
                                                "--build-arg \"TON_LABS_ABI_IMAGE=${G_images['ton-labs-abi']}\" " + 
                                                "--build-arg \"TON_EXECUTOR_IMAGE=${G_images['ton-executor']}\" " +
                                                "--build-arg \"TON_SDK_IMAGE=${G_images['ton-sdk']}\""
                        }
                    }
                    stages {
                        stage('Build') {
                            environment {
                                X86_64_UNKNOWN_LINUX_GNU_OPENSSL_LIB_DIR='/usr/lib/x86_64-linux-gnu'
                                X86_64_UNKNOWN_LINUX_GNU_OPENSSL_INCLUDE_DIR='/usr/include/openssl'
                                X86_64_UNKNOWN_LINUX_GNU_OPENSSL_DIR='/usr/bin/openssl'
                                OPENSSL_DIR='/usr/bin/openssl'
                            }
                            steps {
                                sshagent([G_gitcred]) {
                                    sh """
                                        cd /tonlabs/TON-SDK/ton_client/platforms/ton-client-web
                                        npm install
                                        node build.js
                                    """
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_build = false }
                                }
                            }
                        }
                        stage('Deploy') {
                            when { 
                                expression {
                                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                                }
                            }
                            steps {
                                script {
                                    sh "cp /tonlabs/TON-SDK/ton_client/platforms/ton-client-web/bin/*.gz ./"
                                    stash allowEmpty: true, includes: '*.gz', name: 'web-bin'
                                    /*
                                    withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                        identity = awsIdentity()
                                        s3Upload \
                                            bucket: 'sdkbinaries.tonlabs.io', \
                                            includePathPattern:'*.gz', path: 'tmp_sdk', \
                                            workingDir:'.'
                                    }
                                    */
                                }
                            }
                            post {
                                failure {
                                    script { G_tsnj_deploy = false }
                                }
                            }
                        }
                    }
					post {
						cleanup {script{cleanWs notFailBuild: true}}
					}                
				}
            }
        }
        stage('Deploy to bucket') {
            when { 
                expression {
                    return GIT_BRANCH == 'master' || GIT_BRANCH ==~ '^PR-[0-9]+' || GIT_BRANCH == "${getVar(G_binversion)}-rc"
                }
            }
            steps {
                script {
                    sh """
                        if [ -e 'toDeploy' ] ; then echo 'Remove...' ; rm -rf toDeploy ; fi
                        mkdir toDeploy
                    """
                    dir('toDeploy') {
                        unstash 'cl-linux-bin'
                        unstash 'cl-darwin-bin'
                        unstash 'cl-windows-bin'
                        unstash 'rn-ios-bin'
                        unstash 'rn-android-bin'
                        unstash 'nj-linux-bin'
                        unstash 'nj-windows-bin'
                        unstash 'nj-darwin-bin'
                        unstash 'web-bin'
                        lock('bucket') {
                            withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                                identity = awsIdentity()
                                s3Upload \
                                    bucket: 'sdkbinaries.tonlabs.io', \
                                    includePathPattern:'*.gz', path: 'tmp_sdk', \
                                    workingDir:'.'
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        failure {
            script {
                lock('bucket') {
                    withAWS(credentials: 'CI_bucket_writer', region: 'eu-central-1') {
                        identity = awsIdentity()
                        s3Delete bucket: 'sdkbinaries.tonlabs.io', path: 'tmp_sdk/'
                    }
                }
            }
        }
    }
}