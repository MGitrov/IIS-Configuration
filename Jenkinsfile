pipeline {
    agent {label "Local-Agent"}

    /*environment {
        REPOSITORY_URL = "https://github.com/MGitrov/IIS-Jenkins-Pipeline"
        MAIN_BRANCH = "main"
        SECONDARY_BRANCH = "new-page"
        PACKAGE_NAME = "WebApp.zip"
        DEPLOY_PATH = "C:\\inetpub\\wwwroot\\"
        WEB_APP_POOL = "DefaultAppPool"
    }*/

    stages {
        stage('Load Environment Variables') {
            steps {
                script {
                    // Read the .env file using PowerShell
                    /*def envContents = powershell(returnStdout: true, script: 'Get-Content .env -Raw')
                    
                    // Parse the contents and set environment variables
                    envContents.split('\r?\n').each { line ->
                        def keyValue = line.split('=', 2)
                        if (keyValue.size() == 2) {
                            def key = keyValue[0].trim()
                            def value = keyValue[1].trim()
                            env."${key}" = value
                            echo "Setting ${key} to ${value}"
                        }
                    }*/

                                        // Read the .env file using PowerShell
                    def envContents = powershell(returnStdout: true, script: '''
                    $envFilePath = ".env"
                    if (Test-Path $envFilePath) {
                        Get-Content $envFilePath -Raw
                    } else {
                        Write-Error "No .env file found."
                    }
                    ''')

                    // Parse the contents and set environment variables
                    envContents.split('\r?\n').each { line ->
                        // Skip empty lines and comments (lines starting with '#')
                        if (!line.trim().startsWith("#") && line.contains("=")) {
                            def keyValue = line.split('=', 2)
                            if (keyValue.size() == 2) {
                                // Clean and trim key and value
                                def key = keyValue[0].trim()
                                def value = keyValue[1].trim()

                                // Validate if the key and value are not empty
                                if (key && value) {
                                    env."${key}" = value
                                    echo "Setting ${key} to ${value}"
                                }
                            }
                        }    
                    }
                }
            }
        }

        stage("Verify environment variables") {
            steps {
                script {
                    powershell '''
                    Write-Host "Repository URL: ${env:REPOSITORY_URL}"
                    Write-Host "Main Branch: ${env:MAIN_BRANCH}"
                    Write-Host "Secondary Branch: ${env:SECONDARY_BRANCH}"
                    Write-Host "Package Name: ${env:PACKAGE_NAME}"
                    Write-Host "Deploy Path: ${env:DEPLOY_PATH}"
                    Write-Host "Web App Pool: ${env:WEB_APP_POOL}"
                    '''
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
                        echo "Creating deployment package: ${env.PACKAGE_NAME}"
                        
                        powershell '''
                        Write-Host "Compressing files from: ${env:WORKSPACE}"
                        Write-Host "Saving to: ${env:PACKAGE_NAME}"
                        #$itemsToCompress = Get-ChildItem -Path ${env:WORKSPACE} -Recurse
                        Get-ChildItem -Path ./* -Recurse | ForEach-Object { Write-Host $_.FullName }

                        Compress-Archive -Path $env:WORKSPACE -DestinationPath $env:PACKAGE_NAME -Force -Verbose
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