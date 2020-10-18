pipeline {
    agent any
    // -----------------------------------------------------------------------------------------------------
    environment {
        pool_name = 'xxl-job-admin'
        git_url = "gitlab.gexj.xyz:89/test"
        app_git_url = "${git_url}/${pool_name}.git"
        mail_list = 'gxjluck@163.com'
    }

    // -----------------------------------------------------------------------------------------------------

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: "5"))
    }

    // -----------------------------------------------------------------------------------------------------

    parameters {
        gitParameter(
            useRepository: "http://gitlab.gexj.xyz:89/test/xxl-job-admin.git",
            branch: '', 
            branchFilter: 'origin/(.*)', 
            defaultValue: 'master',
            listSize: "3", 
            name: 'GIT_REVISION', 
            quickFilterEnabled: true, 
            selectedValue: 'NONE', 
            sortMode: 'DESCENDING_SMART', 
            tagFilter: '*', 
            type: 'PT_BRANCH', 
            description: 'Please select a branch or tag to build')

        choice(name: 'ENVIRONMENT',description: 'Please select environment',choices: 'test\ndev')

        booleanParam(name: 'SonarSanner', defaultValue: true, description: 'SonarSanner开关，默认启用，如果跳过代码扫描请去掉打勾。')
		
        booleanParam(name: 'Junit', defaultValue: true, description: 'Junit开关，默认启用，如果跳过请去掉打勾。')
        
        booleanParam(name: 'Allure', defaultValue: true, description: 'Allure开关，默认启用，如果跳过请去掉打勾。')

        booleanParam(name: 'gitTag', defaultValue: true, description: 'gitTag开关，默认启用，如果跳过请去掉打勾。')
                    
        booleanParam(name: 'Deploy', defaultValue: true, description: 'Deploy开关，默认启用，如果跳过请去掉打勾。')
    }

    // -----------------------------------------------------------------------------------------------------

    stages {
        stage ("1.初始化") {
            steps {
                script {
		    wrap([$class: 'BuildUser']) {
                        buildName "${BUILD_NUMBER}-$BUILD_USER-${GIT_REVISION}"
                        }

                    // -----------------------------------------------------------------------------------------------------
                    node ("jnlp") {
                        stage('2.Git克隆') {
                            deleteDir()

                            checkout([$class: 'GitSCM',
                            branches: [[name: "$GIT_REVISION"]],
                            userRemoteConfigs: [[credentialsId: 'git',url: "http://$app_git_url" ]]])
                        
                            if (!GIT_REVISION)  { error "Require git revision"    }
                        }
                    

                    // -----------------------------------------------------------------------------------------------------
			if (params.SonarSanner == true) {
		            stage('3.代码扫描') {
				container('jnlp'){
				println "Scanner Code"
			        env.APP_PROJECT     = JOB_NAME.split('_')[0]
			        env.APP_NAME        = JOB_NAME.split('_')[1] 
			
			        withSonarQubeEnv('sonar') {
				sh 'sonar-scanner \
				-Dsonar.projectKey=$APP_NAME \
				-Dsonar.projectName=$APP_NAME \
				-Dsonar.projectVersion=$GIT_REVISION \
                                -Dsonar.sourceEncoding=UTF-8 \
				-Dsonar.projectBaseDir=. \
				-Dsonar.language=java \
				-Dsonar.sources=. \
				-Dsonar.java.binaries=.'
			}	
			// sleep(10)
			// timeout(1) {
			// def qualitygate = waitForQualityGate('sonar') //需要安装SonarQube Quality Gate插件，并配置
			// 	if (qualitygate.status != "OK") {
			// 		error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
			// 	} 
					//  }
				}
			    }
	                }
		
		// ===================================================================================================

		if (params.Junit == true) {
			stage('4.单元测试') {
		            container('jnlp'){
		            println "Junit"
		            sh 'mvn test -Dmaven.test.failure.ignore=false'
		            findText(textFinders: [textFinder(regexp: 'There are test failures',alsoCheckConsoleOutput: true,buildResult: 'ABORTED')])
		            
		            publishHTML([allowMissing: true, 
		            alwaysLinkToLastBuild: false, 
		            keepAll: false, 
		            reportDir: './target', 
		            reportFiles: 'surefire-report.html', 
		            reportName: '测试报告', 
		            reportTitles: '测试报告'
		            ])
		            
		            jacoco()
		            }
			}
                 }

                // ===================================================================================================

                if (params.Allure == true){
                    container ("jnlp"){
                        stage('5.报告生成') {
                            allure(includeProperties: false, 
                            jdk: '', 
                            properties: [],
                            reportBuildPlolicy: 'ALWAYS',
                            results: [[path: "./target"]]
                            )
                        }
                    }
                }


                // ===================================================================================================

                stage('6.代码编译') {
                    container('jnlp'){
                    println "Build"
                    sh 'mvn -U clean package -Dmaven.test.skip:truedependency:tree'
                    }
                }
                

                // -----------------------------------------------------------------------------------------------------

                withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'passwd', usernameVariable: 'user')]) {
                    stage('7.制作镜像') {
                        container('jnlp'){
                        println "build image"
                        env.APP_PROJECT     = JOB_NAME.split('_')[0]
                        env.B_TIME          = sh(returnStdout: true, script: "date +%Y%m%d%H%M%S")
                        sh '''docker build -t harbor.gexj.xyz:4433/${APP_PROJECT}/${pool_name}:${GIT_REVISION}-${BUILD_NUMBER} . -f ./$pool_name/Dockerfile
                       	docker login -u $user -p $passwd harbor.gexj.xyz:4433
                       	docker push harbor.gexj.xyz:4433/${APP_PROJECT}/${pool_name}:${GIT_REVISION}-${BUILD_NUMBER}'''
                       	}
                    }
		}

                 // -----------------------------------------------------------------------------------------------------


                if (params.gitTag == true){
                    container ("jnlp") {
                        stage('8.Git标签') {
                            env.git_user_email = "test@test.com"
                            env.git_user_name  = "test"
                            deleteDir()
                            withCredentials([usernamePassword(credentialsId: 'git', passwordVariable: 'passwd', usernameVariable: 'user')]) {
                                
                                sh("git clone \"http://$user:$passwd@$app_git_url\"")
                                dir("$pool_name") {
                                    sh """
                                        git config user.email "${git_user_email}"
                                        git config user.name "${git_user_name}"
                                        git checkout "${GIT_REVISION}"
                                        git tag -a -m "test_image_tag"  "${JOB_NAME}_${BUILD_NUMBER}_tag"
                                        git push origin ${JOB_NAME}_${BUILD_NUMBER}_tag
                                    """
                                }
                            }
                        }
                    }
                }

                // -----------------------------------------------------------------------------------------------------

       		if (params.Deploy == true){
       		    stage('9.应用部署') {
       			container('jnlp'){
                            println "deploy to kubernetes with helm"
                            env.git_user_email = "test@test.cmo"
                            env.git_user_name  = "test"
                            env.APP_PROJECT     = JOB_NAME.split('_')[0]
                            env.APP_NAME        = JOB_NAME.split('_')[1]
                            env.B_TIME          = sh(returnStdout: true, script: "date +%Y%m%d%H%M%S")
                            env.APP_NAMESPACE   = "default"

                        if (env.ENVIRONMENT == 'test'){
                            sh "helm -n ${APP_NAMESPACE} upgrade -i ${APP_NAME} ${APP_NAME}/helm-charts/${APP_NAME}/ \
                            --set image.repository=harbor.gexj.xyz:4433/${APP_PROJECT}/${APP_NAME} \
                            --set image.tag=${GIT_REVISION}-${BUILD_NUMBER} \
                            --set env=${env.ENVIRONMENT} \
                            --set namespace=${APP_NAMESPACE}" 
                        }else if (env.ENVIRONMENT == 'dev'){
                            sh "helm -n ${APP_NAMESPACE} upgrade -i ${APP_NAME} ${APP_NAME}/helm-charts/${APP_NAME}/ \
                            --set image.repository=harbor.gexj.xyz:4433/${APP_PROJECT}/${APP_NAME} \
                            --set image.tag=${GIT_REVISION}-${BUILD_NUMBER} \
                            --set env=${env.ENVIRONMENT} \
                            --set namespace=${APP_NAMESPACE}" 
                        }else{
                           println "Your choice envrionment not exist."
                          }
       		       }
                   }
                }
             }
             // -----------------------------------------------------------------------------------------------------

                }
            }
        }
	// -----------------------------------------------------------------------------------------------------
	}
	
	post {
            always{
	        script{
	             println "Finished. Sendmail to mail_list."
	        }
	
	    }

	}
}
