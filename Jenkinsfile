def p = [:]
node {
    checkout scm
    p = readProperties interpolate: true, file: 'ci/pipeline.properties'
}

pipeline {
	agent none

	triggers {
		pollSCM 'H/10 * * * *'
		upstream(upstreamProjects: "spring-data-commons/main", threshold: hudson.model.Result.SUCCESS)
	}

	options {
		disableConcurrentBuilds()
		buildDiscarder(logRotator(numToKeepStr: '14'))
	}

	stages {
		stage("test: baseline (main)") {
			when {
				anyOf {
					branch 'main'
					not { triggeredBy 'UpstreamCause' }
				}
			}
			agent {
				label 'data'
			}
			options { timeout(time: 30, unit: 'MINUTES') }

			environment {
				DOCKER_HUB = credentials("${p['docker.credentials']}")
				ARTIFACTORY = credentials("${p['artifactory.credentials']}")
			}

			steps {
				script {
					docker.withRegistry(p['docker.registry'], p['docker.credentials']) {
						docker.image(p['docker.java.main.image']).inside(p['docker.java.inside.docker']) {
							sh "docker login --username ${DOCKER_HUB_USR} --password ${DOCKER_HUB_PSW}"
							sh 'PROFILE=none ci/verify.sh'
							sh "ci/clean.sh"
						}
					}
				}
			}
		}

		stage("Test other configurations") {
			when {
				allOf {
					branch 'main'
					not { triggeredBy 'UpstreamCause' }
				}
			}
			parallel {
				stage("test: baseline (next)") {
					agent {
						label 'data'
					}
					options { timeout(time: 30, unit: 'MINUTES') }

					environment {
						DOCKER_HUB = credentials("${p['docker.credentials']}")
						ARTIFACTORY = credentials("${p['artifactory.credentials']}")
					}

					steps {
						script {
							docker.withRegistry(p['docker.registry'], p['docker.credentials']) {
								docker.image(p['docker.java.next.image']).inside(p['docker.java.inside.docker']) {
									sh "docker login --username ${DOCKER_HUB_USR} --password ${DOCKER_HUB_PSW}"
									sh 'PROFILE=java11 ci/verify.sh'
									sh "ci/clean.sh"
								}
							}
						}
					}
				}

				stage("test: baseline (LTS)") {
					agent {
						label 'data'
					}
					options { timeout(time: 30, unit: 'MINUTES') }

					environment {
						DOCKER_HUB = credentials("${p['docker.credentials']}")
						ARTIFACTORY = credentials("${p['artifactory.credentials']}")
					}

					steps {
						script {
							docker.withRegistry(p['docker.registry'], p['docker.credentials']) {
								docker.image(p['docker.java.lts.image']).inside(p['docker.java.inside.docker']) {
									sh "docker login --username ${DOCKER_HUB_USR} --password ${DOCKER_HUB_PSW}"
									sh 'PROFILE=java11 ci/verify.sh'
									sh "ci/clean.sh"
								}
							}
						}
					}
				}
			}
		}

		stage('Release to artifactory') {
			when {
				anyOf {
					branch 'main'
					not { triggeredBy 'UpstreamCause' }
				}
			}
			agent {
				label 'data'
			}
			options { timeout(time: 20, unit: 'MINUTES') }

			environment {
				ARTIFACTORY = credentials("${p['artifactory.credentials']}")
			}

			steps {
				script {
					docker.withRegistry(p['docker.registry'], p['docker.credentials']) {
						docker.image(p['docker.java.main.image']).inside(p['docker.java.inside.basic']) {
							sh 'MAVEN_OPTS="-Duser.name=jenkins -Duser.home=/tmp/jenkins-home" ./mvnw -s settings.xml -Pci,artifactory -Dmaven.repo.local=/tmp/jenkins-home/.m2/spring-data-elasticsearch-non-root ' +
									'-Dartifactory.server=https://repo.spring.io ' +
									"-Dartifactory.username=${ARTIFACTORY_USR} " +
									"-Dartifactory.password=${ARTIFACTORY_PSW} " +
									"-Dartifactory.staging-repository=libs-snapshot-local " +
									"-Dartifactory.build-name=spring-data-elasticsearch " +
									"-Dartifactory.build-number=${BUILD_NUMBER} " +
									'-Dmaven.test.skip=true clean deploy -U -B'
						}
					}
				}
			}
		}
		stage('Publish documentation') {
			when {
				branch 'main'
			}
			agent {
				label 'data'
			}
			options { timeout(time: 20, unit: 'MINUTES') }

			environment {
				ARTIFACTORY = credentials("${p['artifactory.credentials']}")
			}

			steps {
				script {
					docker.withRegistry(p['docker.registry'], p['docker.credentials']) {
						docker.image(p['docker.java.main.image']).inside(p['docker.java.inside.basic']) {
							sh 'MAVEN_OPTS="-Duser.name=jenkins -Duser.home=/tmp/jenkins-home" ./mvnw -s settings.xml -Pci,distribute -Dmaven.repo.local=/tmp/jenkins-home/.m2/spring-data-elasticsearch-non-root ' +
									'-Dartifactory.server=https://repo.spring.io ' +
									"-Dartifactory.username=${ARTIFACTORY_USR} " +
									"-Dartifactory.password=${ARTIFACTORY_PSW} " +
									"-Dartifactory.distribution-repository=temp-private-local " +
									'-Dmaven.test.skip=true clean deploy -U -B'
						}
					}
				}
			}
		}
	}

	post {
		changed {
			script {
				slackSend(
						color: (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger',
						channel: '#spring-data-dev',
						message: "${currentBuild.fullDisplayName} - `${currentBuild.currentResult}`\n${env.BUILD_URL}")
				emailext(
						subject: "[${currentBuild.fullDisplayName}] ${currentBuild.currentResult}",
						mimeType: 'text/html',
						recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider']],
						body: "<a href=\"${env.BUILD_URL}\">${currentBuild.fullDisplayName} is reported as ${currentBuild.currentResult}</a>")
			}
		}
	}
}
