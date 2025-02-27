#!/usr/bin/env groovy

pipeline {
    agent {
        kubernetes {
            defaultContainer 'python3'
            yamlFile 'KubernetesPod.yaml'
            retries 2
        }
    }
            
    stages {
        stage('SCM Get Code') {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/piky/lets-quiz.git']]])
            }
        }

        stage('Installing packages') {
            steps {
                script {
                    sh '/usr/local/bin/python3 -m pip install -r playbooks/files/requirements_test.txt'
                }
            }
        }
        
        stage('Static Code Checking') {
            steps {
                script {
                    sh 'find . -name \\*.py | xargs /usr/local/bin/python3 -m pylint --load-plugins=pylint_django -f parseable | tee pylint.log'
                    recordIssues(
                        tool: pyLint(pattern: 'pylint.log'),
                        failTotalHigh: 10,
                    )
                }
            }
        }
        stage('Build and Tag') {
            steps {
                withCredentials([gitUsernamePassword(credentialsId: 'gitea_repo', gitToolName: 'git')]) {
                    sh """
                        git reset --hard HEAD
                        git checkout main
                        git pull origin main --force --rebase
                        git fetch --tags --all --prune
                        git config --replace-all user.name ${env.GIT_USERNAME}
                        git config --replace-all user.email ${env.GIT_USERNAME}
                        cd app && /usr/local/bin/python3 -m bumpversion --config-file setup.cfg --allow-dirty --verbose minor --list > build_vars.env

                    """
                    script {
                        def build_vars = readProperties file: 'app/build_vars.env'
                        env.newPkgVersion = build_vars.new_version
                        env.pkgVersion = build_vars.current_version
                        echo " CURRENT - ${pkgVersion}"
                        echo " NEW  - ${newPkgVersion}"
                    }
                    sh """
                        git tag --force v${newPkgVersion}
                        git add .
                        git commit -m"Bump version from  v${pkgVersion} to v${newPkgVersion}"
                        git push --force origin main v${newPkgVersion}
                    """
                }
            }
        }
    }

}
