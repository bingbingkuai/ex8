pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: gradle
                    image: gradle
                    command:
                    - sleep
                    args:
                    - 99d
                    volumeMounts:
                    - name: shared-storage
                      mountPath: /mnt        
                  - name: kaniko
                    image: gcr.io/kaniko-project/executor:debug
                    command:
                    - sleep
                    args:
                    - 9999999
                    volumeMounts:
                    - name: shared-storage
                      mountPath: /mnt
                    - name: kaniko-secret
                      mountPath: /kaniko/.docker
                  restartPolicy: Never
                  volumes:
                  - name: shared-storage
                    persistentVolumeClaim:
                      claimName: jenkins-pv-claim
                  - name: kaniko-secret
                    secret:
                        secretName: dockercred
                        items:
                        - key: .dockerconfigjson
                          path: config.json
            """
        }
    }
    
    stages {

    stage('Checkout code and prepare environment') {
      steps {
        git url: 'https://github.com/bingbingkuai/chp5.git', branch: 'master'
        sh '''
          cd Chapter08/sample1
          chmod +x gradlew
        '''
      }
    }

    stage('Run pipeline against a gradle project - main branch') {
      when {
       branch 'main'
      }
      steps {
         echo 'On main branch'
            container('gradle') {
                sh '''
                cd Chapter08/sample1
                ./gradlew build
                ./gradlew test
                ./gradlew jacocoTestCoverageVerification
                mv ./build/libs/calculator-0.0.1-SNAPSHOT.jar /mnt
                '''
            }
      }
    }

    stage('Run pipeline against a gradle project - other branches') {
      when {
         not { branch 'main' }
      }
      steps {
        echo ' NOT on main branch'
        script {
          try {
                sh '''
                  cd Chapter08/sample1
                  ./gradlew build
                  ./gradlew checkstyleMain
                  cp ./build/libs/calculator-0.0.1-SNAPSHOT.jar /app/calculator.jar
                  mv ./build/libs/calculator-0.0.1-SNAPSHOT.jar /mnt
                '''
          } catch (Exception E) {
                echo 'Oh no. Test FAIL!!!'
          }
        }
          //HTML publisher plugin
          publishHTML (target: [
              reportDir: 'Chapter08/sample1/build/reports/checkstyle',
              reportFiles: 'checkstyle.xml',
              reportName: "JaCoCo Checkstyle"
          ])
      }
    }
    }

    post {
    success {
        script {
        // Check for branch using environment variable or expression
        if (env.BRANCH_NAME == 'main' ) {
            echo 'Create container repository on main'
            container('kaniko') {
            sh '''
                echo "FROM openjdk:8-jre" > Dockerfile
                echo "COPY ./calculator-0.0.1-SNAPSHOT.jar app.jar" >> Dockerfile
                echo "ENTRYPOINT [\"java\", \"-jar\", \"app.jar\"]" >> Dockerfile
                mv /mnt/calculator-0.0.1-SNAPSHOT.jar .

                # Build and push image using Kaniko
                /kaniko/executor --context `pwd` --destination bingkuai8/calculator:1.0
            '''
            }
        } else {
            echo 'Skipping container build (not on main)'
        }

        if ( env.BRANCH_NAME == 'feature') {
            echo 'Create container repository on feature branch'
            container('kaniko') {
            sh '''
                echo "FROM openjdk:8-jre" > Dockerfile
                cp ./build/libs/calculator-0.0.1-SNAPSHOT.jar /app/calculator.jar
                echo "WORKDIR /app"
                echo "COPY ./calculator-0.0.1-SNAPSHOT.jar app.jar" >> Dockerfile
                echo "ENTRYPOINT [\"java\", \"-jar\", \"app.jar\"]" >> Dockerfile
                mv /mnt/calculator-0.0.1-SNAPSHOT.jar .

                # Build and push image using Kaniko
                /kaniko/executor --context `pwd` --destination bingkuai8/calculator-feature:0.1
            '''
            }
        } else {
            echo 'Skipping container build (not on feature branch)'
        }


        }
    }
    }
}
