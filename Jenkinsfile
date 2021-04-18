def VERSION="${VERSIONTOBUILD}"
pipeline {
    agent {
        label 'master'
    }
    environment {
        //общие переменные для сред
        DeployDir = './charts/laravel/'
        BuildDir = './src'
        repotobuild = 'https://github.com/IsieIam/laravel.git'
        branchtobuild = 'Release'
    }
    stages {
        stage('CloneApp') {
            when {
                allOf {
                    environment name: 'BUILD', value: 'yes'
                }
            }
            steps {
                echo " === Clone App"
                dir("src"){
                    git url: "${repotobuild}", branch: "${branchtobuild}", credentialsId: ''
                }
            }
        }
        stage ('GitVersionTask') {
            when {
                allOf {
                    environment name: 'VERSIONTOBUILD', value: 'gitver'
                }
            }
            steps {
                script{
                    dir("src"){
                        sh "gitversion /output buildserver /nofetch /config"
                        def properties = readProperties file: 'gitversion.properties'
                        VERSION="${properties['GitVersion_MajorMinorPatch']}-${properties['GitVersion_CommitsSinceVersionSource']}"
                    }
                }
            }
        }
        stage('Build') {
            when {
                allOf {
                    environment name: 'BUILD', value: 'yes'
                }
            }
            steps {
                script {
                    sh "docker build --no-cache -t ${ImagesRepoUri}:${VERSION} ${BuildDir}"
                }
            }
        }
        stage('Upload') {
            when {
                allOf {
                    environment name: 'BUILD', value: 'yes'
                }
            } 
            steps {
                script {
                    sh "docker push ${ImagesRepoUri}:${VERSION}"
                }
            }
        }
        stage("Cleanup") {
            when {
                allOf {
                    environment name: 'BUILD', value: 'yes'
                }
            } 
            steps {
                script {
                    sh "docker rmi ${ImagesRepoUri}:${VERSION}"
                }
            }
        }
        stage('Deploy Stage') {
            steps {
                script {
                    cd ${DeployDir}
                    helm ls
                    helm upgrade --install larka . -f values.yaml -n stage
                }
            }
        }
        stage('Tests') {
            steps {
                echo "some tests" 
            }
        }
        stage('Deploy Prod') {
            steps {
                script {
                    cd ${DeployDir}
                    helm ls
                    helm upgrade --install larka . -f values.yaml -n prod
                }
            }
        }
    }
    post {
        always {
            echo 'Cleanup'
        }
    }
}
