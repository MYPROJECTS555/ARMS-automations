# ARMS-automations


es. Based on your ARMS structure, here is a Jenkinsfile for Windows Jenkins Agent + SQL Server + sqlcmd -E.

It will:

Checkout GitHub code
Find changed .sql files
Backup the existing stored procedure
Execute only the changed SQL file
Stop if execution fails
pipeline {
    agent any

    environment {
        DB_SERVER = 'YOUR-SQL-SERVER'
        DB_NAME   = 'ARMS'
        BACKUP_DIR = 'C:\\Jenkins\\DB_Backups'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Find Changed SQL Files') {
            steps {
                script {
                    def changedFiles = bat(
                        script: 'git diff --name-only HEAD~1 HEAD',
                        returnStdout: true
                    ).trim()

                    def sqlFiles = changedFiles
                        .split('\r?\n')
                        .findAll { it.toLowerCase().endsWith('.sql') }

                    if (sqlFiles.isEmpty()) {
                        echo "No SQL files changed."
                        currentBuild.result = 'NOT_BUILT'
                        return
                    }

                    env.CHANGED_SQL_FILES = sqlFiles.join('|')

                    echo "Changed SQL files:"
                    sqlFiles.each {
                        echo it
                    }
                }
            }
        }

        stage('Backup and Execute SQL') {
            steps {
                script {

                    if (!env.CHANGED_SQL_FILES) {
                        echo "No SQL files to execute."
                        return
                    }

                    def sqlFiles = env.CHANGED_SQL_FILES.split('\\|')

                    sqlFiles.each { sqlFile ->

                        echo "Processing: ${sqlFile}"

                        /*
                         * Example:
                         * ArmsOPSOthers/Schema Objects/Schemas/auto/
                         * Programmability/Stored Procedures/
                         * AutoRosterLoop.proc.sql
                         */

                        def fileName = sqlFile.split('/')[-1]

                        // Remove .proc.sql
                        def objectName = fileName
                            .replace('.proc.sql', '')

                        // Get schema from path
                        def pathParts = sqlFile.split('/')

                        def schemaIndex = pathParts.findIndexOf {
                            it.equalsIgnoreCase('Schemas')
                        }

                        def schemaName =
                            pathParts[schemaIndex + 1]

                        echo "Schema: ${schemaName}"
                        echo "Procedure: ${objectName}"

                        bat """
                        if not exist "${BACKUP_DIR}" mkdir "${BACKUP_DIR}"

                        sqlcmd -E ^
                          -S "${DB_SERVER}" ^
                          -d "${DB_NAME}" ^
                          -Q "IF OBJECT_ID(N'[${schemaName}].[${objectName}]', N'P') IS NOT NULL BEGIN SELECT OBJECT_DEFINITION(OBJECT_ID(N'[${schemaName}].[${objectName}]')) AS Definition END" ^
                          -o "${BACKUP_DIR}\\${schemaName}_${objectName}_before_build_${BUILD_NUMBER}.sql"
                        """

                        echo "Backup completed for ${schemaName}.${objectName}"

                        bat """
                        sqlcmd -E ^
                          -S "${DB_SERVER}" ^
                          -d "${DB_NAME}" ^
                          -b ^
                          -i "${sqlFile}"
                        """

                        echo "Successfully executed: ${sqlFile}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Database deployment completed successfully."
        }

        failure {
            echo "Database deployment FAILED."
        }
    }
}
Your actual flow

For this file:

ArmsOPSOthers/
└── Schema Objects/
    └── Schemas/
        └── auto/
            └── Programmability/
                └── Stored Procedures/
                    └── AutoRosterLoop.proc.sql

Jenkins identifies:

Schema     = auto
Procedure  = AutoRosterLoop

Then:

SQL Server
   ↓
Check auto.AutoRosterLoop
   ↓
Backup existing definition
   ↓
Execute AutoRosterLoop.proc.sql
   ↓
If successful → SUCCESS
If failed      → FAILURE
One important improvement
