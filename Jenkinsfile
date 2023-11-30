/**
* Parameters can be sent via build parameters, instead of changing the code.
* Use the same variable name to set the build parameters.
* List of parameters that can be passed
* appName='devops-demo-web-app'
* deployableName = 'PROD-US'
* componentName="web-app-v1.1"
* collectionName="release-1.0"
* exportFormat ='yaml'
* configFilePath = "k8s/helm/values.yml"
* exporterName ='returnAllData-now' 
* exporterArgs = ''
*/

pipeline {
    environment {
        buildArtifactsPath = "build_artifacts/${currentBuild.number}"
        validationResultsPath = ""
    }
    agent any
    /**
    * Jenkins pipline related variables
    */
    stages {
        // Initialize pipeline
        stage('Initialize') {
            steps {
                script {
                    dockerImageName = "kekaichinose/web-app"

                    /**
                    * DevOps Config App related information
                    */
                    appName = 'PaymentDemo'
                    deployableName = 'Production'
                    componentName = "web-api-v1.0"
                    collectionName = "release-1.0"
                    /**
                    * Configuration File information to be uploaded
                    */ 
                    configFileFormat = 'yaml'
                    configFilePath = "k8s/helm/values.yml"
                    /**
                    * Devops Config exporter related information
                    */
                    exporterName = 'returnAllData-now' 
                    exporterArgs = ''
                    exportFormat = 'yaml'
                    /**
                    * Jenkins variables declared to be used in pipeline
                    */
                    exportFileName = "${buildArtifactsPath}/export_file-${appName}-${deployableName}-${currentBuild.number}.${exportFormat}"
                    changeSetId = ""
                    dockerImageTag = ""
                    snapshotName = ""
                    snapshotObject = ""
                    isSnapshotCreated = false
                    isSnapshotValidateionRequired = false
                    isSnapshotPublisingRequired = false
                    skipChange = true
                    
                    buildNumberArtifact = "grefId123"

                    /**
                    * Checking for parameters
                    */
                    if(params) {
                        echo "setting values from build parameter"
                        if(params.appName) {
                            appName = params.appName;
                        }
                        if(params.deployableName) {
                            deployableName = params.deployableName
                        }
                        if(params.componentName) {
                            componentName = params.componentName
                        }
                        if(params.collectionName) {
                            collectionName = params.collectionName
                        }
                        if(params.configFileFormat) {
                            configFileFormat = params.configFileFormat
                        }
                        if(params.configFilePath) {
                            configFilePath = params.configFilePath
                        }
                        if(params.exporterName) {
                            exporterName = params.exporterName
                        }
                        if(params.exporterArgs) {
                            exporterArgs = params.exporterArgs
                        }
                        if(params.exportFormat) {
                            exportFormat = params.exportFormat
                        }
                        if(params.skipChange) {
                            skipChange = params.skipChange
                        }
                    }
                }
                echo """---- Build Parameters ----
                applicationName: ${appName}
                namePath: ${componentName}
                configFile: ${configFilePath}
                dataFormat: ${configFileFormat}
                """
            }
        }
            
        // Build and publish application image
        stage('Build') {      
            steps {
                checkout scm    
                echo "scm checkout successful"
                
                script {
                    dockerImageTag = env.BUILD_NUMBER
                    dockerImageNameTag = "${dockerImageName}" + ":" + "${dockerImageTag}"

                    snDevopsArtifactPayload = '{"artifacts": [{"name": "' + dockerImageName + '",  "version": "' + "${dockerImageTag}" + '", "semanticVersion": "' + "0.1.${dockerImageTag}"+ '","repositoryName": "' + dockerImageName+ '"}, ],"stageName":"Build image","branchName": "main"}'  ;
                    echo "Docker image artifact: ${dockerImageNameTag} "
                    echo "snDevopsArtifactPayload: ${snDevopsArtifactPayload} "

                    snDevOpsArtifact(artifactsPayload:snDevopsArtifactPayload)
                }
            }
        } 
            
        // Validate code and config data
        stage('Validate') {
            parallel {
                // Validate configuration data changes
                stage('Config') {
                    stages('Config Steps') {
                        // Upload configuration data to DevOps Config
                        stage('Upload, Validate, & Publish') {
                            steps {
                                //sh "echo updating configfile with build number to allow rerun without config file changes"
                                //sh "sed -i 's/${buildNumberArtifact}/${BUILD_NUMBER}/g' ${configFilePath}"
                                sh "echo uploading and auto-validating configuration file: ${configFilePath}"
                                script {
                                    changeSetResults = snDevOpsConfig(
                                        applicationName: "${appName}",
                                        target: 'component',
                                        namePath: "${componentName}",
                                        configFile: "${configFilePath}",
                                        dataFormat: "${configFileFormat}",
                                        autoCommit: 'true',
                                        autoValidate: 'true',
                                        autoPublish: 'true',
                                        isValidated: 'true',
                                        continueWithLatest: 'true',
                                        markFailed: 'true'
                                    )

                                    echo "Snapshots generated, validated, and published: ${changeSetResults}"

                                    def changeSetResultsObject = readJSON text: changeSetResults

                                    changeSetResultsObject.each {
                                        snapshotName = it.name
                                        snapshotObject = it
                                    }

                                    validationResultsPath = "${snapshotName}_${currentBuild.projectName}_${currentBuild.number}*.xml"
                                }
                            }
                        }

                        // Export published snapshot to be used by downstream deployment tools
                        stage('Export') {
                            steps {
                                script {
                                    echo "Exporting config data for App: ${appName}, Deployable: ${deployableName}, Exporter: ${exporterName} "
                                    echo "Export file name ${exportFileName}"
                                    // create build artifacts dir if not created yet
                                    sh "mkdir -p ${buildArtifactsPath}"
                                    
                                    echo "<<<<<<<<< Starting config data export >>>>>>>>"
                                    exportResponse = snDevOpsConfigExport(
                                            applicationName: "${appName}",
                                            snapshotName: "${snapshotObject.name}",
                                            deployableName: "${deployableName}",
                                            exporterFormat: "${exportFormat}",
                                            fileName: "${exportFileName}",
                                            exporterName: "${exporterName}",
                                            exporterArgs: "${exporterArgs}"
                                    )
                                    echo "RESPONSE FROM EXPORT : ${exportResponse}"
                                }
                            }
                        }
                    }
                }

                // Validate application code changes (SIMULATED)
                stage('Code') { 
                    stages {
                        stage('jUnit Test'){ 
                            steps {
                                echo "Running unit tests..."
                            }
                        }
                        
                        stage('SonarQube analysis') {
                            steps {
                                echo "Running code quality analysis..."
                            }
                        }
                    }    
                }
            }
        }

        // Deploy configuration data to UAT environment
        stage('UAT Deployment') {
            steps {
                sleep(time:5,unit:"SECONDS")
            }
        }
        
        // Run functional tests
        stage ('Functional Testing') {
            parallel {      
                stage('Selenium API') { 
                    steps {
                        echo "Selenium API..2..3..4"
                        sleep(time:5,unit:"SECONDS")
                        echo "Selenium API..2..3..4"
                    }
                }
                stage('Selenium UI') {
                    steps {
                        echo "Selenium UI..2..3..4"
                        sleep(time:7,unit:"SECONDS")
                        echo "Selenium API..2..3..4"
                    }
                }
            }
        }
        
        // Submit change management review
        stage('Change Management') {
            steps {
                //node('built-in')
                script {
                    // Enable change acceleration
                    if(skipChange) {
                        echo "<<< Skip DevOps Change >>>"
                    } else {
                        echo "DevOps Change - trigger change request"
                        snDevOpsChange(
                                applicationName: "${appName}",
                                snapshotName: "${snapshotName}"
                        )
                        // ALTERNATE - CR with application service details
                        /*echo "DevOps Change - trigger change request"
                        snDevOpsChange(changeRequestDetails: """{
                                "setCloseCode": false,
                                "attributes": {
                                    "category": "DevOps",
                                    "priority": "3",
                                    "cmdb_ci": {
                                        "name": "Servers - PaymentDemo - Production"
                                    },
                                    "business_service": {
                                        "name": "PaymentDemo_Production_1"
                                    }
                                }
                        }""")
                        */
                    }
                }     
            }
        }

        // Deploy application code and configuration data to production environment
        stage('Deploy to Production') {
                steps {
                    script {
                            echo "Show exported config data from file name ${exportFileName}"
                            echo " ++++++++++++ BEGIN OF File Content ***************"
                            sh "cat ${exportFileName}"
                            echo " ++++++++++++ END OF File content ***************"
                            echo "Exported config data handed off to deployment tool"
                            echo "********************** BEGIN Deployment ****************"
                            echo "Applying docker image ${dockerImageNameTag}"
                            echo "********************** END Deployment ****************"
                    }
                }
        }
    }
    // NOTE: attach policy validation results to run (if the snapshot fails validation)
    post {
        always {
            // create tests dir
            sh "mkdir -p ${buildArtifactsPath}/tests"
            // move policy validation results to build artifacts folder
            //sh "mv ${validationResultsPath} ${buildArtifactsPath}/tests/${validationResultsPath}"
            // attach policy validation results
            echo ">>>>> Displaying Test results <<<<<"
            //junit testResults: "${buildArtifactsPath}/tests/${validationResultsPath}", skipPublishingChecks: true
        }
    }
}
