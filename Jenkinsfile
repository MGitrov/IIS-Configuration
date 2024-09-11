pipeline {
    agent {label "Local-Agent"}

    /*environment {
        // Default values in case ".env" file fails to load
        REPOSITORY_URL = "https://github.com/MGitrov/IIS-Jenkins-Pipeline"
        MAIN_BRANCH = "main"
        SECONDARY_BRANCH = "new-page"
        PACKAGE_NAME = "WebApp.zip"
        DEPLOY_PATH = "C:\\inetpub\\wwwroot\\"
        WEB_APP_POOL = "DefaultAppPool"
    }*/

    stages {
        stage('Load .env File') {
            steps {
                script {
                    // Read the .env file content using PowerShell and return the key-value pairs
                    def envVars = powershell(returnStdout: true, script: '''
                    $envFilePath = ".env"

                    if (Test-Path $envFilePath) {
                        Get-Content $envFilePath | ForEach-Object {
                            $parts = $_ -split "="
                            if ($parts.Count -eq 2) {
                                $envName = $parts[0].Trim()
                                $envValue = $parts[1].Trim()
                                Write-Output "$envName=$envValue"
                            }
                        }
                    } else {
                        Write-Host ".env file not found."
                        return ""
                    }
                    ''').trim()

                    // Prepare the environment variables for 'withEnv'
                    def envList = []
                    envVars.split('\n').each { line ->
                        if (line) {
                            envList.add(line.trim()) // Add the key=value pair to the environment list
                        }
                    }

                    // Apply the environment variables using 'withEnv'
                    withEnv(envList) {
                        echo "Loaded environment variables: ${env.REPOSITORY_URL}, ${env.MAIN_BRANCH}"
                    }
                }
            }
        }

        stage("Verify environment variables") {
            steps {
                script {
                    echo "Repository URL: ${env.REPOSITORY_URL}"
                    echo "Main Branch: ${env.MAIN_BRANCH}"
                    echo "Secondary Branch: ${env.SECONDARY_BRANCH}"
                    echo "Package Name: ${env.PACKAGE_NAME}"
                    echo "Deploy Path: ${env.DEPLOY_PATH}"
                    echo "Web App Pool: ${env.WEB_APP_POOL}"
                }
            }
        }

        stage("Checkout 'main' branch") {
            steps {
                script {
                    echo "Building main branch..."
                    checkout([$class: "GitSCM", branches: [[name: "*/${env.MAIN_BRANCH}"]],
                              userRemoteConfigs: [[url: "${env.REPOSITORY_URL}"]]
                    ])
                }

                powershell "ls" // Ensures that Jenkins pulled all the files.
            }
        }

        stage("Fetch 'newpage' file from 'new-page' Branch") {
            when {
                branch "new-page"
            }
            
            steps {
                script {
                    echo "Fetching files from ${env.BRANCH_NAME} branch..." /* BRANCH_NAME is a Jenkins environment variable that
                    is used to identify the branch that triggered the build. */

                    /* "2> $null" means that any non-fatal errors from the "git fetch" command will be ignored. */
                    powershell '''
                    git fetch origin ${env:BRANCH_NAME} 2> $null
                    git checkout origin/${env:BRANCH_NAME} -- newpage.html
                    ls
                    '''
                }
            }
        }

        stage("Deployment package creation") { /* This stage packages the application files into a ".zip" file that will later be deployed 
        to the IIS web server. */
            steps {
                    script {
                        powershell '''
                        Compress-Archive -Path ./* -DestinationPath $env:PACKAGE_NAME -Force -Verbose
                        '''
                    }
            }
        }

        stage("Verify ZIP file creation") {
            steps {
                powershell '''
                if (Test-Path "$env:WORKSPACE\\$env:PACKAGE_NAME") {
                    Write-Host "ZIP file created successfully: $env:PACKAGE_NAME"
                    ls
                } else {
                    Write-Error "ZIP file was not created"
                }
                '''
            }
        }

        stage("ZIP file contents verification") {
            steps {
                echo "Verifying contents of the ZIP file..."
                powershell '''
                Add-Type -AssemblyName System.IO.Compression.FileSystem
                $zipFile = [System.IO.Compression.ZipFile]::OpenRead("$env:WORKSPACE\\$env:PACKAGE_NAME")
                $zipFile.Entries | ForEach-Object { $_.FullName }
                $zipFile.Dispose()
                '''
            }
        }

        stage("Deployment to IIS web server") {
            steps {
                echo "Deploying to IIS web server..."
                script {
                    powershell '''
                    Start-Process "C:\\Program Files\\IIS\\Microsoft Web Deploy V3\\msdeploy.exe" -ArgumentList @(
                        "-verb:sync",
                        "-source:package='$env:WORKSPACE\\$env:PACKAGE_NAME'",
                        "-dest:contentPath='$env:DEPLOY_PATH',computerName='localhost'",
                        "-enableRule:DoNotDeleteRule"
                    ) -Wait -NoNewWindow
                    '''
                }
            }
        }

        stage("Recycling web app pool") {
            steps {
                echo "Recycling ${WEB_APP_POOL} web app pool..."
                script {
                    powershell '''
                    Restart-WebAppPool -Name $env:WEB_APP_POOL
                    '''
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}