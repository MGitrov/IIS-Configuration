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
                    echo "Loading environment variables from .env file..."

                    // Use PowerShell to read the .env file and capture key-value pairs
                    /*def envVars = powershell(returnStdout: true, script: '''
                    $envFilePath = ".env"
                    $output = @()
                    if (Test-Path $envFilePath) {
                        Get-Content $envFilePath | ForEach-Object {
                            if ($_ -match "=") {
                                $key, $value = $_ -split "="
                                $key = $key.Trim()
                                $value = $value.Trim()
                                if (![string]::IsNullOrEmpty($key)) {
                                    $output += "$key=$value"
                                    Write-Host "Setting environment variable: $key=$value"
                                }
                            }
                        }
                    } else {
                        Write-Host ".env file not found."
                    }
                    return $output -join "`n"
                    ''').trim()

                    // Parse and set the environment variables in the Jenkins context
                    envVars.split('\n').each { line ->
                        def (key, value) = line.split('=')
                        env[key] = value.trim()
                    }*/

                    powershell '''
                    $envFilePath = "$env:WORKSPACE\\.env"
                    Get-Content $envFilePath | ForEach-Object {
                        if ($_ -match "^\s*([^#][^=]+)=(.*)$") {
                            [System.Environment]::SetEnvironmentVariable($matches[1], $matches[2])
                        }
                    }
                    '''

                    // Print the loaded variables for debugging
                    echo "Repository URL: ${env.REPOSITORY_URL}"
                    echo "Main Branch: ${env.MAIN_BRANCH}"
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