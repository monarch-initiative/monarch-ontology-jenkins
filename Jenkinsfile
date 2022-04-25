pipeline {
    agent { label 'large-worker' }
    // In additional to manual runs, trigger somewhere at midnight to
    // give us the max time in a day to get things right without
    // disrupting people.
    triggers {
    // Nightly, between 8pm-12:59pm PDT.
    cron('H H(20-23) 1-31 * *')
    }
    environment {
    ///
    /// Automatic run variables.
    ///

    // Acquire dates and day to beginning of run.
    START_DATE = sh (
        script: 'date +%Y-%m-%d',
        returnStdout: true
    ).trim()

    START_DAY = sh (
        script: 'date +%A',
        returnStdout: true
    ).trim()

    ///
    /// Internal run variables.
    ///

    // What is the file "namespace" of the ontology--used for
    // finding artifacts.
    ONTOLOGY_FILE_HINT = 'monarch'
    // Ontology repo information.
    TARGET_ONTOLOGY_BRANCH = 'mo-orig-sri'
    TARGET_ONTOLOGY_URL = 'https://github.com/monarch-initiative/monarch-ontology.git'
    // The people to call when things go bad or go well. It is a
    // comma-space "separated" string.
    TARGET_ADMIN_EMAILS = 'nicolas.matentzoglu@gmail.com'
    TARGET_SUCCESS_EMAILS = 'nicolas.matentzoglu@gmail.com'
    // This variable should typically be 'TRUE', which will cause
    // some additional basic checks to be made. There are some
    // very exotic cases where these check may need to be skipped
    // for a run, in that case this variable is set to 'FALSE'.
    WE_ARE_BEING_SAFE_P = 'TRUE'
    // Control make to get through our loads faster if
    // possible.
    MAKECMD = 'make'
    // Control the ROBOT environment.
    ROBOT_JAVA_ARGS = '-Xmx50G'
    JAVA_OPTS = '-Xmx50G'
    JAVAARGS = '-Xmx50G'
    }
    options{
    //timestamps()
    buildDiscarder(logRotator(numToKeepStr: '3'))
    }
    stages {
    // Very first: check branch sanity and pause for a minute to
    // give a chance to cancel; clean the workspace before use.
    stage('Ready and clean') {
        steps {
        // Give us a minute to cancel if we want.
        //sleep time: 1, unit: 'MINUTES'
        cleanWs()
        }
    }
    stage('Initialize') {
        steps {
        // Start preparing environment.
        sh 'env > env.txt'
        sh 'echo $BRANCH_NAME > branch.txt'
        sh 'echo "$BRANCH_NAME"'
        sh 'cat env.txt'
        sh 'cat branch.txt'
        sh 'echo $START_DAY > dow.txt'
        sh 'echo "$START_DAY"'
        archiveArtifacts artifacts: "env.txt"
        }
    }
    stage('Produce ontology') {
        agent {
            docker {
                label 'large-worker'
                image 'obolibrary/odkfull'
                // Reset Jenkins Docker agent default to original
                // root.
                args '-u root:root'
                        alwaysPull true
            }
        }
        steps {
        // Create a relative working directory and setup our
        // data environment.
        dir('.') {
            //sh 'rm -rf human-phenotype-ontology-master'
            //sh 'wget https://github.com/obophenotype/human-phenotype-ontology/archive/master.zip -O master.zip'
            //sh 'unzip master.zip'
            //sh 'ls -l'
           
            checkout([
                $class: 'GitSCM',
                extensions: [[$class: 'CheckoutOption', timeout: 120]] + [[$class: 'CloneOption', timeout: 120, shallow: true, noTags:true, depth: 1]], gitTool: 'Default', 
                userRemoteConfigs: [[url: 'https://github.com/monarch-initiative/monarch-ontology.git']],
            ])
            
            sh 'ls -l'
            //sh 'ls -l human-phenotype-ontology-master'
            //sh 'chown -R jenkins:jenkins human-phenotype-ontology-master'
            //sh 'ls -l human-phenotype-ontology-master'

            dir('.') {
                    retry(1){
                        sh 'ls -l'
                        sh 'mkdir -p build'
                        sh "sed -i 's/--no-logging//g' Makefile"
                        sh "sed -i 's|data.monarchinitiative.org/owl/|data.monarchinitiative.org/monarch/202109/owl/|g' monarch.owl"
                        sh 'make all -B'
                    }
            }

            // Move the products to somewhere "safe".
            archiveArtifacts artifacts: "build/*",
            onlyIfSuccessful: true

            // Now that the files are safely away onto skyhook for
            // debugging, test for the core dump.
            script {
            if( WE_ARE_BEING_SAFE_P == 'TRUE' ){

                def found_core_dump_p = fileExists 'target/core_dump.owl'
                if( found_core_dump_p ){
                error 'ROBOT core dump detected--bailing out.'
                }
            }
            }
        }
        }
    }
    // stage('Archive') {
    //     when { anyOf { branch 'release'; branch 'snapshot'; branch 'master' } }
    //     steps {
    //     // Stanza to push to real file server.
    //     }
    // }
    }
    post {
    // Let's let our people know if things go well.
    success {
        script {
        echo "There has been a successful run of the ${env.BRANCH_NAME} pipeline."
        }
    }
    // Let's let our internal people know if things change.
        changed {
        echo "There has been a change in the ${env.BRANCH_NAME} pipeline."
    }
    // Let's let our internal people know if things go badly.
    failure {
        echo "There has been a failure in the ${env.BRANCH_NAME} pipeline."
        // mail bcc: '', body: "There has been a pipeline failure in ${env.BRANCH_NAME}. Please see: ${env.JOB_DISPLAY_URL}", cc: '', from: '', replyTo: '', subject: "Pipeline FAIL for ${env.ONTOLOGY_FILE_HINT} on ${env.BRANCH_NAME}", to: "${TARGET_ADMIN_EMAILS}"
        }
    }
}


