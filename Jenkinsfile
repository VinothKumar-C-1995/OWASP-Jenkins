stage('SonarQube Scan') {

    steps {

        withSonarQubeEnv('SonarQube') {

            sh '''
            mvn sonar:sonar
            '''

        }

    }

}
