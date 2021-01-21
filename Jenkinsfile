pipeline {
	agent {
		label 'docker-multiarch-big'
	}
	options {
		buildDiscarder(logRotator(numToKeepStr: '5'))
		disableConcurrentBuilds()
		ansiColor('xterm')
	}
	environment {
		IMAGE        = 'alpine-nginx-full'
		BUILDX_NAME  = "${IMAGE}_${GIT_BRANCH}"
		BRANCH_LOWER = "${BRANCH_NAME.toLowerCase().replaceAll('/', '-')}"
		// Software versions; OpenResty does not support Lua >= 5.2
		OPENRESTY_VERSION = '1.19.3.1'
		LUA_VERSION       = '5.1.5'
		LUAROCKS_VERSION  = '3.3.1'
	}
	stages {
		stage('Environment') {
			parallel {
				stage('Master') {
					when {
						branch 'master'
					}
					steps {
						script {
							env.BASE_TAG                = 'latest'
							env.BUILDX_PUSH_TAGS        = "-t docker.io/ezodoo/${IMAGE}:${BASE_TAG}"
							env.BUILDX_PUSH_TAGS_NODE   = "-t docker.io/ezodoo/${IMAGE}:node"
							env.BUILDX_PUSH_TAGS_GOLANG = "-t docker.io/ezodoo/${IMAGE}:golang"
						}
					}
				}
				stage('Other') {
					when {
						not {
							branch 'master'
						}
					}
					steps {
						script {
							// Defaults to the Branch name, which is applies to all branches AND pr's
							env.BASE_TAG                = "github-${BRANCH_LOWER}"
							env.BUILDX_PUSH_TAGS        = "-t docker.io/ezodoo/${IMAGE}:${BASE_TAG}"
							env.BUILDX_PUSH_TAGS_NODE   = "${BUILDX_PUSH_TAGS}-node"
							env.BUILDX_PUSH_TAGS_GOLANG = "${BUILDX_PUSH_TAGS}-golang"
						}
					}
				}
			}
		}
		stage('Base Build') {
			environment {
				BUILDX_NAME = "${IMAGE}_${GIT_BRANCH}_base"
			}
			steps {
				withCredentials([usernamePassword(credentialsId: 'ezodoo-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
					sh "docker login -u '${duser}' -p '${dpass}'"
					sh "./scripts/buildx --push ${BUILDX_PUSH_TAGS}"
				}
			}
		}
		stage('Other Builds') {
			parallel {
				stage('Golang') {
					environment {
						BUILDX_NAME  = "${IMAGE}_${GIT_BRANCH}_golang"
					}
					steps {
						sh 'sed -i "s/BASE_TAG/${BASE_TAG}/g" Dockerfile.golang'
						withCredentials([usernamePassword(credentialsId: 'jc21-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
							sh "docker login -u '${duser}' -p '${dpass}'"
							sh "./scripts/buildx --push -f Dockerfile.golang ${BUILDX_PUSH_TAGS_GOLANG}"
						}
					}
				}
				stage('Node') {
					environment {
						BUILDX_NAME  = "${IMAGE}_${GIT_BRANCH}_node"
					}
					steps {
						sh 'sed -i "s/BASE_TAG/${BASE_TAG}/g" Dockerfile.node'
						withCredentials([usernamePassword(credentialsId: 'jc21-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
							sh "docker login -u '${duser}' -p '${dpass}'"
							sh "./scripts/buildx --push -f Dockerfile.node ${BUILDX_PUSH_TAGS_NODE}"
						}
					}
				}
			}
		}
		stage('PR Comment') {
			when {
				allOf {
					changeRequest()
					not {
						equals expected: 'UNSTABLE', actual: currentBuild.result
					}
				}
			}
			steps {
				script {
					def comment = pullRequest.comment("""Docker Image for build ${BUILD_NUMBER} is available on [DockerHub](https://cloud.docker.com/repository/docker/ezodoo/${IMAGE}) as:

- `jc21/${IMAGE}:github-${BRANCH_LOWER}`
- `jc21/${IMAGE}:github-${BRANCH_LOWER}-node`
- `jc21/${IMAGE}:github-${BRANCH_LOWER}-golang`
""")
				}
			}
		}
	}
	post {
		success {
			juxtapose event: 'success'
			sh 'figlet "SUCCESS"'
		}
		failure {
			juxtapose event: 'failure'
			sh 'figlet "FAILURE"'
		}
		unstable {
			juxtapose event: 'unstable'
			sh 'figlet "UNSTABLE"'
		}
	}
}
