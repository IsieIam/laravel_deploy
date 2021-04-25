// Для запуска пайплайн ожидает три параметра на вход:
// VERSIONTOBUILD - string parameter: значение по умолчанию gitver - версия приложения/тег докер образа для сборки или установки.
// BUILD - choise parameter: yes/no - пересобирать приложение или нет. При этом если выбираете no, то просто попробует задеплоиться версия указанная в параметре VERSIONTOBUILD.
// additional_args - string parameter - может быть пустым или доп списком параметров к параметру helm --set, для примера на ходу переопределить внешний адрес сервиса.
// для примера, обратить внимание на зпт в начале: ,laravel-nginx.ingress.host=84.252.131.52.nip.io

// часть этапов выполняться будут в зависимости от значений параметров - есть build нам не нужен то этапы связанные с самим docker образом пропускаются

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
    // Забираем исходныйй код приложения из его репо по адресу репо и ветки для забора
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
    // Определяем версию приложения через gitbersiontask, которая сама запускается в докер контейнере (это дает возможность не ставить полноценную версию на worker ноду jenkins)
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
          // вот именно здесь определяется версия нашего образа/приложения компновкой собранных фактов о репо приложения утилитой gitverion task
          VERSION="${properties['GitVersion_MajorMinorPatch']}-${properties['GitVersion_CommitsSinceVersionSource']}"
        }
      }
    }
    // билдим образ - все просто
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
    // пушим образ в регистри
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
    // удаляем собранный образ с worker ноды Jenkins
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
    // подготавливаем helm - устанавливаем зависимости
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
    // деплоим наше приложение из helm- чарта на stage
    stage('Deploy Stage') {
      steps {
        script {
          sh "helm upgrade --install larka-stage ${DeployDir}/. -f ${DeployDir}/values.yaml --set laravel-nginx.image.php.tag=${VERSION}${additional_args} -n stage --wait"
        }
      }
    }
    // этап для запуска автотестов
    stage('Tests') {
      steps {
        echo "some tests"
      }
    }
    // деплоим наше приложение из helm- чарта на prod
    stage('Deploy Prod') {
      steps {
        script {
          sh "helm upgrade --install larka ${DeployDir}/. -f ${DeployDir}/values.yaml --set laravel-nginx.image.php.tag=${VERSION}${additional_args} -n prod --wait"
        }
      }
    }
  }
  // post действие
  post {
    always {
      echo 'Cleanup'
    }
  }
}
