pipeline {
    agent {
        node {
            label 'Deepak_System'
        }
    }
    stages {
        stage("Intial Setup") {
            steps {
                script{
                echo "-----------------------------------------"
                echo 'Setting the DisplayName'
				echo "-----------------------------------------"
				currentBuild.displayName = "CCA Test Pipeline: Adapter ${Adapter_Type}"
				echo "-----------------------------------------"
                echo 'Setting the Description'
				echo "-----------------------------------------"
				currentBuild.description = "Current Job is running against Firmware ${CCA_FIRMWARE} in ${RUN_MODE} mode"
                }
                
            }
        }
        stage('Fetch the TS') {
            steps {
                echo "-----------------------------------------"
                echo 'Clean The Env'
				echo "-----------------------------------------"
				cleanWs()
				sh 'rm -rf *'
			    echo "-----------------------------------------"
                echo 'Fetching the Test script from Gerrit'
				echo "-----------------------------------------"
                //git branch: 'master', credentialsId: 'Test-Team-Jenkins-Gerrit', url: 'ssh://0b8632649@fwgerrit.boeblingen.de.ibm.com:29418/clt-crypto-test-cca'
                git branch: 'main', credentialsId: 'Test-Team-Jenkins-Gerrit', url: 'git@github.ibm.com:Crypto-hsm/crypto-cca-firmware-tests.git'

            }
        }
        stage('FETCH THE LATEST RC BUILD') {
            steps {
				echo "-----------------------------------------"
                echo "Fetching the latest Test build"
				echo "-----------------------------------------"
                sh '''
                pwd
                sed -i 's/CCA_ADAPTER=""/CCA_ADAPTER='"$Adapter_Type"'/' pull_4769_rc_code.sh
                sed -i 's/CCA_FIRMWARE=""/CCA_FIRMWARE='"$CCA_FIRMWARE"'/' pull_4769_rc_code.sh
                echo "pull_4769_rc_code.sh script has been updated"
                echo "Copy the pull_4769_rc_code.sh script"
                ./pull_4769_rc_code.sh
                '''
            }
        }
        stage('ENV SETUP AND START TESTS') {
            steps {
                script {
                    try {
                        echo "-----------------------------------------"
                        echo "Setting the env variable and Start the regression"
            			echo "-----------------------------------------"
                        sh '''
                        ROOT_DIR=$(printenv JOB_NAME)
                        rm -rf hashing_in_process
                        cp Automation/regression.sh .
                        cp ../default_run.fle .
                        ./regression.sh $Adapter_Type $CCA_ADAPTER_SELECTION $S390_DOMAIN_NUM $CSU_DEFAULT_ADAPTER $ROOT_DIR $TEST_MODE $RUN_MODE $TEST_CASE $CCA_ADAPTER_CERT
                        '''
                    } catch (err) {
                        echo err.getMessage()
                    }
                    
                }
            }
            post {
                always {
                    sh '''
                    cp Automation/csv_2_html.py .
                    rm -rf *warn_*.csv
                    CSV_FILE_NAME=$(ls -l |find . -name '*.csv')
				    rm -rf ../log/* | mkdir -p ../log | cp $CSV_FILE_NAME ../log
                    python csv_2_html.py $CSV_FILE_NAME $BUILD_URL
                    '''
                    echo "-----------------------------------------"
				    echo "archiving the artifacts"
				    echo "-----------------------------------------"
				    archiveArtifacts artifacts: '*.err, *_err.cas, *.lnx, *.csv', followSymlinks: false
                }
            }
        }
        stage('Generate the HTML Report') {
            steps {
                echo "Geneate the HTML Report"
                sh 'mkdir -p Test_Report; mv crypto_report.html Test_Report'
                 // publish html
                publishHTML([
							allowMissing: true,
							alwaysLinkToLastBuild: true,
							includes: '**/*.html',
							keepAll: true,
							reportDir: 'Test_Report',
							reportFiles: 'crypto_report.html',
							reportName: 'CCA Test Report',
							reportTitles: 'Crypto Test Report'
                    ])   
            }
        }
        stage('Clean the env once the build is completed') {
            steps {
                echo "-----------------------------------------"
                echo "Clean the workspace and Archive the build generated artifacts "
				echo "-----------------------------------------"
				sh '''
				rm -rf regression.sh
				rm -rf pull_4769_rc_code.sh
				rm -rf $Adapter_Type*
				'''
				cleanWs(patterns: [[pattern: '*.csv,  *.lnx, *.err,  *_err.cas', type: 'INCLUDE']])
				
            }
        }
        stage('Update the DataBase') {
            when {
                environment name: 'Update_database', value: 'Yes'
            }
            steps { 
                script {
            		def remote = [:]
                    remote.name = 'test'
                    remote.host = '9.47.84.76'
                    remote.user = 'root'
                    remote.password = 'passw0rd'
                    remote.allowAnyHosts = true
                    echo "-----------------------------------------"
                    echo "Copy all the script and Update the database for Grafana Server"
        			echo "-----------------------------------------"
        			def csv_file = sh (script: "echo \$(ls | find * -name '*.csv')", returnStdout: true).trim()
        			def FW_Version = sh (script: "echo \$(ls | find * -name '*.csv' | head -c 6)", returnStdout: true).trim()
        			echo "-------------------------------"
                    echo "${csv_file}"
                    echo "${FW_Version}"
                    sshPut remote: remote, from: "${csv_file}", into: '/root/tmp', override: true
                    sshCommand remote: remote, command: "ls -lrt /root/tmp"
                    sshPut remote: remote, from: "Automation/csv_to_influx/", into: '/root/tmp', override: true
                    sshPut remote: remote, from: "${csv_file}", into: '/root/tmp/csv_to_influx', override: true
                    sshCommand remote: remote, command: "ls -lrt /root/tmp/csv_to_influx/"
                    sshCommand remote: remote, command: "chmod +x -R /root/tmp/csv_to_influx/"
                    sshCommand remote: remote, command: "ls -lrt /root/tmp/csv_to_influx/"
                    sshCommand remote: remote, command: "hostname; cd  /root/tmp/csv_to_influx/; /usr/bin/python3.6 write_ini.py ${csv_file} ${csv_file}  crypto_db admin passw0rd ${BUILD_NUMBER} 4770  ${FW_Version} ${RUN_MODE} ${TEST_MODE}"
                    sshCommand remote: remote, command: "cat /root/tmp/csv_to_influx/config.ini"
                    sshCommand remote: remote, command: "hostname; cd  /root/tmp/csv_to_influx/; /usr/bin/python3.6 csv_parser_influxdb.py"
                }
            }
        }
        
    }
    post {
        always {
            script {
                JOB_STATUS="${currentBuild.getCurrentResult()}"
                sh """
                curl -X POST -F channel=cat_test_slack -F text="Jenkins Pipeline Build ID: ${BUILD_ID} is ${JOB_STATUS}. Details: Job: $JOB_NAME with Adapter: ${Adapter_Type} and Firmware: ${CCA_FIRMWARE} in ${RUN_MODE} mode - Build URL: ${BUILD_URL}" https://slack.com/api/chat.postMessage -H "Authorization:  Bearer xoxb-359950653876-YuXG5jhOWRIfMR08QM7INLgb"
                """
            }
        }
    }
}
