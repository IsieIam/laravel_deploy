def VERSION="${VERSIONTOBUILD}"
pipeline {
  agent {
    label 'master'
  }
  environment {
    //общие переменные для сред
    // имя образа
    registry = "isieiam/laravel"
    // каталог с основным чартом
    DeployDir = './charts/laravel'
    // каталог куда мы заливаем исходный код приложния
    BuildDir = './src'
    // адрес репо с которого мы берем исходный код
    repotobuild = 'https://github.com/IsieIam/laravel.git'
    // ветка релиза для автоматического билда - подразумевается что с репа настроен хук при изменения в этой ветке на начало деплоя 
    branchtobuild = 'Release'
    // путь до kubeconfig на дженке - так делать не очень красиво, сделано для простоты примера, правильней кубконфиг хранить в кредах дженка
    KUBECONFIG = '/var/lib/jenkins/.kube/kubeconfig'
    // id кредов для push в докерхаб
    registryCredential = 'dockerhub_id'
    dockerImage = ''
  }
  stages {
    stage('CloneApp') {
      when {
        allOf {
          environment name: 'BUILD', value: 'yes'
          environment name: 'VERSIONTOBUILD', value: 'gitver'
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
          withDockerContainer(args: "--entrypoint='' -v ${WORKSPACE}/src:/repo", image: "gittools/gitversion:5.3.5-linux-alpine.3.10-x64-netcoreapp3.1") {
            sh "cd /repo && /tools/dotnet-gitversion /repo /output buildserver /nofetch /config"
          }
          def properties = readProperties file: 'src/gitversion.properties'
          VERSION="${properties['GitVersion_MajorMinorPatch']}-${properties['GitVersion_CommitsSinceVersionSource']}"
        }
      }
    }
    stage('Build') {
      when {
        allOf {
          environment name: 'BUILD', value: 'yes'
          environment name: 'VERSIONTOBUILD', value: 'gitver'
        }
      }
      steps {
        script {
          //sh "docker build --no-cache -t ${ImagesRepoUri}:${VERSION} ${BuildDir}"
          dir("src"){
            dockerImage = docker.build registry + ":$VERSION"
          }
        }
      }
    }
    stage('Upload') {
      when {
        allOf {
          environment name: 'BUILD', value: 'yes'
          environment name: 'VERSIONTOBUILD', value: 'gitver'
        }
      }
      steps {
        script {
          //sh "docker push ${ImagesRepoUri}:${VERSION}"
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage("Cleanup") {
      when {
        allOf {
          environment name: 'BUILD', value: 'yes'
          environment name: 'VERSIONTOBUILD', value: 'gitver'
        }
      }
      steps {
        script {
          sh "docker rmi $registry:${VERSION}"
        }
      }
    }
    stage('prepare helm') {
      steps {
        script{
          sh '''
            cd ${DeployDir}
            helm dep update
          '''
        }
      }
    }
    stage('Deploy Stage') {
      steps {
        script {
          sh "helm upgrade --install larka-stage ${DeployDir}/. -f ${DeployDir}/values.yaml --set laravel-nginx.image.php.tag=${VERSION}${additional_args} -n stage --wait"
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
          sh "helm upgrade --install larka ${DeployDir}/. -f ${DeployDir}/values.yaml --set laravel-nginx.image.php.tag=${VERSION}${additional_args} -n prod --wait"
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
