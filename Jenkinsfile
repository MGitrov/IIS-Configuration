pipeline {
    agent {label "Local-Agent"}

    parameters {
        string(name: "REPOSITORY_URL", defaultValue: "https://github.com/MGitrov/IIS-Jenkins-Pipeline.git", description: "GitHub repository to checkout from")
        string(name: "MAIN_BRANCH", defaultValue: "main", description: "Main branch")
        
    }

    stages {
        stage("Load environmet variables") {
            steps {
                script {
                    // Reads the ".env" file.
                    def envVariables = powershell(returnStdout: true, script: "Get-Content .env -Raw")
                    
                    // Parses the contents of the ".env" file, and sets the environment variables in the pipeline.
                    envVariables.split("\r?\n").each { line ->
                        def keyValue = line.split("=", 2)
                        if (keyValue.size() == 2) {
                            def key = keyValue[0].trim()
                            def value = keyValue[1].trim()
                            env."${key}" = value
                            echo "Setting ${key} to ${value}"
                        
                            if (key == REPOSITORY_URL) {
                                params.REPOSITORY_URL = value
                            }
                            
                            if (key == MAIN_BRANCH) {
                                params.MAIN_BRANCH = value
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

        stage("Checkout the main branch") {
            steps {
                script {
                    echo "Building ${params.MAIN_BRANCH} branch..."
                    checkout([$class: "GitSCM", branches: [[name: "*/${params.MAIN_BRANCH}"]],
                              userRemoteConfigs: [[url: "${params.REPOSITORY_URL}"]]
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

                        # Excludes the "Configuration" directory and ".gitmodules" file during compression.
                        $itemsToCompress = Get-ChildItem -Path $env:WORKSPACE -Recurse | Where-Object {
                        $_.FullName -notlike "*\\Configuration*" -and $_.FullName -notlike "*.gitmodules"
                        }

                        # Lists the files to be compressed.
                        $itemsToCompress | ForEach-Object { Write-Host "Including: $($_.FullName)" }

                        Compress-Archive -Path $itemsToCompress.FullName -DestinationPath $env:PACKAGE_NAME -Force -Verbose
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