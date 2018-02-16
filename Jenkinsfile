pipeline {

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 3, unit: 'HOURS')
    }

    agent none
    tools {
        maven 'M3'
        jdk 'JDK8'
        git 'GIT2' // This tools doesn't not modify the PATH should do it manualy after
        //org.jenkinsci.plugins.ansible.AnsibleInstallation 'ANSIBLE2' not working for the moment
    }

    stages {

        stage('Prepare') {
            agent { label 'master' }
            steps {
                echo "Path without git : ${env.PATH}"
                script {
					env.PATH = "/apps/git/bin:" + env.PATH
					def pom = readMavenPom file: 'pom.xml'
					env.development_version = pom.version
					env.release_version = env.development_version.replace("-SNAPSHOT", ".${currentBuild.number}")
					env.artifact_id = pom.artifactId
					env.git_url = sh(returnStdout: true, script: 'git config remote.origin.url').trim()
					env.jacoco_plugin = "org.jacoco:jacoco-maven-plugin:0.7.9"
					env.release_plugin = "org.apache.maven.plugins:maven-release-plugin:2.5.3"
					env.sonar_plugin = "org.sonarsource.scanner.maven:sonar-maven-plugin:3.2"
					env.repo_id = pom.distributionManagement.repository.id
					env.repo_url = pom.distributionManagement.repository.url
				}
			sh 'printenv'
            }
        }

        stage('Build and Test') {
            agent { label 'master' }
            steps {
                echo 'Launch build'
                sh 'mvn -B ${jacoco_plugin}:prepare-agent -Dmaven.test.failure.ignore package'

                echo 'Store Test Results'
                junit '**/target/surefire-reports/TEST-*.xml'
            }
        }


        stage('Analyse with SonarQube') {
            agent { label 'master' }
            steps {
                script {
                    env.proxy_line = env.http_proxy ? ' -Dhttp.proxyHost='+ new URI(env.http_proxy).host+' -Dhttp.proxyPort='+new URI(env.http_proxy).port : ''
                    env.sonar_project_version = (env.release_version + '-' + env.BRANCH_NAME).take(90)
                }
                withSonarQubeEnv('default-sonar') {
                sh 'mvn -B $proxy_line ${sonar_plugin}:sonar -Dsonar.scm.disabled=True -Dsonar.projectVersion="$sonar_project_version"'
                }
                script {
                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status == "ERROR") {
                        error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
                    } else {
                        if(qualitygate.status == "WARN") {
                            currentBuild.result='UNSTABLE'
                        }
                        echo "Pipeline quality gate coverage : ${qualitygate.status}"
                    }
                }
            }
        }

        stage('Prepare Release') {
            agent { label 'master' }
            when {
                anyOf { branch 'master'; branch 'stable-*' }
            }
            steps {
                echo "build nb : ${currentBuild.number}"
                echo "releaseVersion     : ${env.release_version}"
                echo "developmentVersion : ${env.development_version}"
                echo "Local maven release without pushing and deploying..."
                sh '''
                mvn -B \
                    -DreleaseVersion=$release_version  \
                    -DdevelopmentVersion=$development_version  \
                    -DpushChanges=false  \
                    -DlocalCheckout=true \
                    -DpreparationGoals=initialize   \
                    "-Darguments=-B -Dmaven.javadoc.skip=true -Dmaven.test.failure.ignore=true -Dmaven.deploy.skip=true" \
                ${release_plugin}:prepare ${release_plugin}:perform
                '''
            }
        }

        stage('Push and Deploy') {
            agent { label 'master' }
            when {
                anyOf { branch 'master'; branch 'stable-*' }
            }
            steps {
                echo "Push the git tag : ${env.artifact_id}-${env.release_version} on http://${env.gitlab_repo} ..."

                withCredentials([[
                        $class: 'UsernamePasswordMultiBinding',
                        credentialsId: 'git-jenkins-credentials',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD']]) {
                    //Follow JENKINS-28335 for a better way to do this
                    sh('git push '+
                            env.git_url.replace("http://", 'http://${GIT_USERNAME}:${GIT_PASSWORD}@')+
                            ' $artifact_id-$release_version')
                }

                echo "Deploy binary to nexus..."
                dir("target/checkout") {
                    sh "mvn -B deploy:deploy-file -DpomFile=pom.xml -Durl=$repo_url -DrepositoryId=$repo_id -Dfile=packaging/target/sugar-packaging-${release_version}.zip -Dpackaging=zip"
                }
            }
        }

        stage('Install DEV') {
            agent { label 'master' }
            when {
                branch 'master'
            }
            steps {
                sshagent (credentials: ['sugar-sugar-pk']) {
                    sh """
ssh -o "StrictHostKeyChecking no" sugar@10.0.100.41 <<'ENDSSH'
sudo bash /home/sugar/install.sh ${env.release_version}
ENDSSH
"""
                }
            }
        }
    
        stage('Run Automation Tests') {
            agent { label 'master' }
            when {
                branch 'master'
            }
            steps {
                sshagent (credentials: ['sugar-sugar-pk']) {
                    sh """
ssh -o "StrictHostKeyChecking no" sugar@10.0.100.41 <<'ENDSSH'
sleep 120
cd /home/sugar/wesa/sugar/TSP-Tests/SUGAR/Test ; mvn -s /home/sugar/wesa/sugar/TSP-Tests/usersettings.xml clean install test -DargLine="-Dpassword=teste -Xmx1g -Duser=testuser" -Dpath=/home/sugar/wesa/sugar/TSP-Tests 
ENDSSH
"""
                }
            }
        }
    }

    post {
        always {
            node('master') {
                deleteDir()
            }
        }
    }
}
