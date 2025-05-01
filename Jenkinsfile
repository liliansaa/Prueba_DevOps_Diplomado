pipeline {
  agent { label 'trabajo' } // Especifica el label del agente donde debe ejecutarse el trabajo

  environment {
    JAVA_HOME = tool name: 'JDK-24', type: 'jdk'
    PATH = "${JAVA_HOME}/bin:${env.PATH}"
    SONARQUBE_URL = 'http://host.docker.internal:9000'
    SONARQUBE_TOKEN = 'sqp_0f775cd820a27679e34a34aeb46c1569602ca981' // Define este secreto en Jenkins
  }

  stages {
     stage('Verificar JAVA_HOME') {
      steps {
        sh 'echo $JAVA_HOME' // Esto imprimirá la ruta de JAVA_HOME para asegurarte de que esté bien configurado
      }
    }
    stage('Checkout Código') {
      steps {
        checkout scm
      }
    }

    stage('Build y Package') {
      tools {
        maven 'mavenlocal'  // El nombre aquí debe coincidir con el que configuraste en Jenkins
      }
      steps {
        sh 'mvn clean package spring-boot:repackage -DskipTests'
      }
    }

    stage('Análisis SonarQube') {
      tools {
        maven 'mavenlocal'  // El nombre aquí debe coincidir con el que configuraste en Jenkins
      }
      steps {
        withSonarQubeEnv('SonarQube') {
          sh """
            mvn sonar:sonar \
              -Dsonar.projectKey=CALCULADORA_API \
              -Dsonar.host.url=${SONARQUBE_URL} \
              -Dsonar.token=${SONARQUBE_TOKEN}
          """
        }
      }
    }

    stage('Instalar Chrome y ChromeDriver') {
      steps {
        sh """
          sudo -s
          apt-get update 
          apt-get install chromium-chromedriver chromium
          ln -s /usr/lib/chromium-browser/chromedriver /usr/local/bin/chromedriver
          chromedriver --version
        """
      }
    }

    stage('Levantar la aplicación') {
      steps {
        sh '''
          nohup java -Dspring.profiles.active=prod -Dheadless -jar target/*.jar & 
          echo $! > pid.txt
        '''
      }
    }

    stage('Esperar aplicación') {
      steps {
        sh '''
          for i in {1..15}; do
            if curl -s http://localhost:8080/; then
              echo "✅ App disponible"
              exit 0
            else
              echo "⏳ Esperando la app... intento $i"
              sleep 5
            fi
          done
          echo "❌ La app no se levantó a tiempo"
          exit 1
        '''
      }
    }

    stage('Probar Endpoint') {
      steps {
        sh 'curl -v "http://localhost:8080/api/calculadora/sumar?a=5&b=7"'
      }
    }

    stage('Tests de Selenium') {
      tools {
        maven 'mavenlocal'  // El nombre aquí debe coincidir con el que configuraste en Jenkins
      }
      steps {
        sh 'mvn test -Dtest=com.ejemplo.calculadora.CalculadoraUITest'
      }
    }

    stage('Mostrar logs') {
      steps {
        sh 'cat nohup.out || echo "No se encontró el archivo de logs"'
      }
    }
  }

  post {
    always {
      echo '🧹 Limpiando recursos...'
      sh '''
        if [ -f pid.txt ]; then
          kill $(cat pid.txt) || echo "No se pudo detener la app"
        fi
      '''
    }
  }
}
