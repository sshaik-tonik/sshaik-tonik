pipeline {
    agent { label 'linux' }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['test', 'uat', 'prod'], description: 'Select the target environment')
        // This is the dynamic file parameter that we will populate after selecting the environment
        choice(name: 'SQL_FILE', choices: [], description: 'Select SQL file from the branch')
    }

    environment {
        DB_CREDENTIALS = credentials('68948b27-0932-42cf-a2ee-bdefbc9c85c1') // Using Git credentials
        DB_HOST_TEST = 'test-db-host'
        DB_HOST_UAT = 'uat-db-host'
        DB_HOST_PROD = 'prod-db-host'
        BRANCH_NAME = ''
        SQL_FILE_PATH = ''
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo "Selected environment: ${params.ENVIRONMENT}"

                    // Map environment to the database host and branch name
                    switch (params.ENVIRONMENT) {
                        case 'test':
                            env.DB_HOST = DB_HOST_TEST
                            env.BRANCH_NAME = 'test'
                            break
                        case 'uat':
                            env.DB_HOST = DB_HOST_UAT
                            env.BRANCH_NAME = 'uat'
                            break
                        case 'prod':
                            env.DB_HOST = DB_HOST_PROD
                            env.BRANCH_NAME = 'prod'
                            break
                        default:
                            error "Invalid environment selected: ${params.ENVIRONMENT}"
                    }

                    // Fetch files from the selected branch to populate the file selection parameter
                    echo "Fetching files from branch: ${env.BRANCH_NAME}"
                    populateFileChoices()
                }
            }
        }

        stage('Fetch Files From Branch') {
            steps {
                script {
                    // Checkout the selected branch (e.g., test, uat, prod)
                    checkout scm: [
                        $class: 'GitSCM',
                        branches: [[name: "*/${BRANCH_NAME}"]],
                        userRemoteConfigs: [[
                            url: 'https://github.com/Tonik-Digital-Bank/poc_test_db_script.git',
                            credentialsId: DB_CREDENTIALS.id
                        ]]
                    ]

                    // After the checkout, ensure that the file parameter is populated dynamically
                    echo "Files from branch ${sshaik_test_db_deploy} fetched."
                }
            }
        }

        stage('List Files') {
            steps {
                script {
                    // List files in the repository for the selected branch
                    def fileList = sh(script: "git ls-tree --name-only HEAD", returnStdout: true).trim().split('\n')
                    echo "Files in branch ${BRANCH_NAME}: ${fileList}"

                    // Update the SQL_FILE parameter choices dynamically
                    def fileChoices = []
                    fileList.each { file ->
                        if (file.endsWith('.sql')) {
                            fileChoices.add(file) // Only add .sql files
                        }
                    }

                    // Set the SQL_FILE parameter choices dynamically
                    currentBuild.description = "Files from ${BRANCH_NAME} branch"
                    currentBuild.setBuildParameter('SQL_FILE', fileChoices)
                }
            }
        }

        stage('Fetch and Display SQL File') {
            steps {
                script {
                    // Ensure the user has selected a valid SQL file
                    if (!params.SQL_FILE) {
                        error "No SQL file selected. Aborting build."
                    }

                    // Checkout the master branch (or the selected branch) again to fetch the correct file content
                    checkout scm: [
                        $class: 'GitSCM',
                        branches: [[name: "*/${BRANCH_NAME}"]],
                        userRemoteConfigs: [[
                            url: 'https://github.com/Tonik-Digital-Bank/poc_test_db_script.git',
                            credentialsId: DB_CREDENTIALS.id
                        ]]
                    ]

                    // Check if the selected SQL file exists in the branch
                    if (!fileExists(params.SQL_FILE)) {
                        error "SQL file ${params.SQL_FILE} not found in branch ${BRANCH_NAME}."
                    }

                    // Display the content of the selected SQL file
                    def sqlContent = readFile(params.SQL_FILE)
                    echo "Contents of SQL file (${params.SQL_FILE}):\n${sqlContent}"
                }
            }
        }

        stage('Execute SQL Script') {
            steps {
                script {
                    echo "Executing SQL file on ${params.ENVIRONMENT} database: ${DB_HOST}"

                    try {
                        // Execute the selected SQL file on the appropriate database
                        sh "mysql --host=${DB_HOST} --user=${DB_CREDENTIALS_USR} --password=${DB_CREDENTIALS_PSW} < ${params.SQL_FILE}"
                        echo "SQL execution completed successfully."
                    } catch (Exception e) {
                        error "SQL execution failed. Error: ${e.message}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline executed successfully."
        }
        failure {
            echo "Pipeline failed. Please check the logs for details."
        }
    }
}

