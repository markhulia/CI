pipeline {
    agent any
    
    stages {
	stage('Download changes') {
	    steps {
		echo 'Pulling changes'
	   }
	}	    
        stage('Build') {
            steps {
                echo 'Building..'
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
            }
        }
    }
}
