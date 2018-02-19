pipeline {

   options {
      disableConcurrentBuilds()
      buildDiscarder(logRotator(numToKeepStr: '10'))
      timeout(time: 3, unit: 'HOURS')
   }

   agent any
   tools {
      maven 'M3'
      jdk 'JDK8'
      git 'GIT2' // This tools doesn't not modify the PATH should do it manualy after
      //org.jenkinsci.plugins.ansible.AnsibleInstallation 'ANSIBLE2' not working for the moment
   }

   stages {

      stage('Prepare') {
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
         steps {
                echo 'Launch build'
                sh 'mvn -B ${jacoco_plugin}:prepare-agent -Dmaven.test.failure.ignore package'
         }
      }

      stage('Analyse with SonarQube') {
         steps {
             dir("api-schema") {
                script {
                    env.proxy_line = env.http_proxy ? ' -Dhttp.proxyHost='+ new URI(env.http_proxy).host+' -Dhttp.proxyPort='+new URI(env.http_proxy).port : ''
                }
                withSonarQubeEnv('default-sonar') {
                    sh 'mvn -B $proxy_line ${sonar_plugin}:sonar -Dsonar.scm.disabled=True -Dsonar.projectVersion="$release_version-$BRANCH_NAME"'
                }
                script {
                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status != "OK") {
                        error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
                    }
                }
             }
         }
      }

      stage('Prepare Release') {
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
                    "-Darguments=-B -Dmaven.javadoc.skip=true -Dmaven.test.skip=true -Dmaven.deploy.skip=true" \
                ${release_plugin}:prepare ${release_plugin}:perform
                '''

                input 'Deploy the binary on nexus and install it on server tc-tsp-ci-scm-srv ?'
         }
      }

      stage('Push and Deploy') {
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
                dir("target/checkout/api-schema") {
                    sh "mvn -B deploy:deploy-file -DpomFile=pom.xml -Durl=$repo_url -DrepositoryId=$repo_id -Dfile=target/sugar-rest-web-api-${release_version}.jar -Dpackaging=jar"
                    sh "mvn -B deploy:deploy-file -DpomFile=pom.xml -Durl=$repo_url -DrepositoryId=$repo_id -Dfile=target/sugar-rest-web-api-${release_version}-sources.jar -Dpackaging=jar -Dclassifier=sources"
                }
                dir("target/checkout/release/api-mock-deployment") {
                    sh "mvn -B deploy:deploy-file -DpomFile=pom.xml -Durl=$repo_url -DrepositoryId=$repo_id -Dfile=target/api-mock-deployment-${release_version}-release.zip -Dpackaging=zip"
                }
                dir("target/checkout/release/api-test-deployment") {
                    sh "mvn -B deploy:deploy-file -DpomFile=pom.xml -Durl=$repo_url -DrepositoryId=$repo_id -Dfile=target/api-test-deployment-${release_version}-release.zip -Dpackaging=zip"
                }
                dir("target/checkout") {
                    sh "mvn -B deploy:deploy-file -DpomFile=pom.xml -Durl=$repo_url -DrepositoryId=$repo_id -Dfile=pom.xml"
                }
         }
      }

   }
   post {
      always {
         deleteDir()
      }
   }
}
