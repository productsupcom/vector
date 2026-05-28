def NAME = 'vector'
def version

pipeline {
    agent { label 'highmem' }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: env.BRANCH_NAME]],
                    extensions: [[$class: 'CloneOption', noTags: false, shallow: false, depth: 0, reference: '']],
                    userRemoteConfigs: scm.userRemoteConfigs,
                ])
            }
        }

        stage('Compute version') {
            steps {
                script {
                    sh 'printenv | sort'
                    def upstreamTag = sh(returnStdout: true, script: "git describe --tags --abbrev=0 || echo v0.0.0").trim()
                    def base = upstreamTag.replaceFirst(/^v/, '')
                    if (env.TAG_NAME) {
                        version = env.TAG_NAME.replaceFirst(/^v/, '')
                    } else if (env.BRANCH_NAME.startsWith('PR-')) {
                        version = "${base}+pr${env.BRANCH_NAME.replace('PR-', '')}.${env.BUILD_ID}"
                    } else {
                        def branchSlug = env.BRANCH_NAME.replaceAll(/[^a-zA-Z0-9]+/, '-').toLowerCase()
                        version = "${base}+${branchSlug}.${env.BUILD_ID}"
                    }
                    sh "echo Building Vector version: ${version}"
                }
            }
        }

        stage('Build vector binary') {
            agent {
                docker {
                    image 'rust:1.92-bookworm'
                    reuseNode true
                    args '-v $HOME/.cargo-vector-cache:/usr/local/cargo/registry'
                }
            }
            steps {
                sh '''
                    apt-get update
                    apt-get install -y --no-install-recommends \
                        pkg-config \
                        libssl-dev \
                        libsasl2-dev \
                        cmake \
                        protobuf-compiler
                '''
                sh '''
                    cargo build --release --no-default-features \
                        --features "api,sources-logstash,sources-vector,sources-internal_metrics,transforms-remap,sinks-redis,sinks-prometheus"
                '''
            }
        }

        stage('Build deb package') {
            agent {
                docker {
                    image 'rust:1.92-bookworm'
                    reuseNode true
                    args '-v $HOME/.cargo-vector-cache:/usr/local/cargo/registry'
                }
            }
            steps {
                sh '''
                    cargo install cargo-deb --version 2.9.3 --locked
                    cargo deb --no-build --deb-version ''' + "${version}-1" + '''
                '''
                sh "ls -la target/debian/*.deb"
                sh "dpkg-deb -I target/debian/*.deb"
                sh "cp target/debian/*.deb ./vector_${version}-1_amd64.deb"
            }
        }

        stage('Publish deb package') {
            when {
                anyOf {
                    buildingTag()
                    branch 'fix/redis-channel-publish-zero-subscribers-v2'
                }
            }
            steps {
                sshagent(credentials: ['jenkins-ssh']) {
                    script {
                        sh "scp -o StrictHostKeyChecking=no vector_${version}-1_amd64.deb root@aptly.productsup.com:/tmp/"
                        sh """
                            ssh -o StrictHostKeyChecking=no root@aptly.productsup.com \
                                "aptly repo add stable /tmp/vector_${version}-1_amd64.deb && \
                                 aptly publish update -passphrase-file='/root/.aptly/passphrase' -batch stable s3:aptly-productsup:debian && \
                                 rm /tmp/vector_${version}-1_amd64.deb"
                        """
                    }
                }
            }
        }
    }

    post {
        cleanup {
            cleanWs deleteDirs: true
        }
    }
}
