// =============================================================================
// Veracode Security Pipeline for Jenkins - Windows Multibranch Version
// =============================================================================
//
// Tested target:
//   https://github.com/veracode/verademo
//   Maven POM: app/pom.xml
//   WAR output: app/target/verademo.war
//
// Required Jenkins credentials:
//   - veracode-api-id        Secret text -> Veracode API ID
//   - veracode-api-key       Secret text -> Veracode API Key
//
// Optional environment variables:
//   - VERACODE_APP_NAME      Override app profile name
//                            If unset, defaults to owner/repo from Git remote URL
//                            Example: ancient-cthulhu/verademo
//   - MAVEN_POM_PATH         Default: app/pom.xml
//   - VERACODE_SOURCE_DIR    Default: app
//
// Branch strategy:
//   feature/*       -> Pipeline Scan, no gate
//   pull request    -> Pipeline Scan with Very High / High gate
//   release/*       -> Veracode Sandbox Scan
//   main            -> Veracode Policy Scan
//
// Notes:
//   - This version is Windows-native and uses PowerShell.
//   - This is designed for a Jenkins Multibranch Pipeline.
//   - Use checkout scm, not hardcoded git branch checkout.
// =============================================================================

pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        disableConcurrentBuilds()

        // Important for Multibranch Pipeline:
        // Jenkins otherwise performs an automatic checkout before stages.
        // We do our own explicit checkout in the Checkout stage.
        skipDefaultCheckout(true)
    }

    environment {
        VERACODE_API_ID  = credentials('veracode-api-id')
        VERACODE_API_KEY = credentials('veracode-api-key')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm

                powershell '''
                    $ErrorActionPreference = "Stop"

                    Write-Host "Repository contents:"
                    Get-ChildItem -Force

                    Write-Host ""
                    Write-Host "Branch:       $env:BRANCH_NAME"
                    Write-Host "Change ID:    $env:CHANGE_ID"
                    Write-Host "Build number: $env:BUILD_NUMBER"
                    Write-Host "Workspace:    $env:WORKSPACE"

                    Write-Host ""
                    Write-Host "Git remote:"
                    git config --get remote.origin.url
                '''
            }
        }

        stage('Resolve Veracode App Name') {
            steps {
                script {
                    def overrideName = env.VERACODE_APP_NAME?.trim()

                    if (overrideName) {
                        env.VERACODE_APP_NAME_RESOLVED = overrideName
                        echo "Veracode Application Profile override: ${overrideName}"
                    } else {
                        def repoUrl = env.GIT_URL ?: env.GIT_URL_1 ?: ""

                        if (!repoUrl?.trim()) {
                            repoUrl = powershell(
                                returnStdout: true,
                                script: '''
                                    $ErrorActionPreference = "Stop"
                                    git config --get remote.origin.url
                                '''
                            ).trim()
                        }

                        def cleaned = repoUrl.trim()

                        // Remove protocol and optional embedded credentials.
                        cleaned = cleaned.replaceFirst(/^https?:\\/\\/([^@\\/]+@)?/, "")

                        // Convert SSH style:
                        // git@github.com:owner/repo.git -> github.com:owner/repo.git
                        cleaned = cleaned.replaceFirst(/^git@/, "")

                        // Remove host prefix:
                        // github.com/owner/repo.git -> owner/repo.git
                        // github.com:owner/repo.git -> owner/repo.git
                        cleaned = cleaned.replaceFirst(/^[^:\\/]+[:\\/]/, "")

                        // Remove trailing .git
                        cleaned = cleaned.replaceFirst(/\\.git$/, "")

                        def parts = cleaned.tokenize("/")
                        def appName

                        if (parts.size() >= 2) {
                            def owner = parts[parts.size() - 2]
                            def repo = parts[parts.size() - 1]
                            appName = "${owner}/${repo}"
                        } else {
                            appName = env.JOB_NAME
                            echo "WARNING: Could not parse owner/repo from Git URL '${repoUrl}'. Falling back to JOB_NAME: ${appName}"
                        }

                        env.VERACODE_APP_NAME_RESOLVED = appName
                        echo "Veracode Application Profile: ${appName}"
                    }
                }
            }
        }

        stage('Build (Maven)') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    $mavenPom = if ($env:MAVEN_POM_PATH) { $env:MAVEN_POM_PATH } else { "app/pom.xml" }

                    Write-Host "Building Maven project: $mavenPom"
                    mvn -B -f "$mavenPom" clean package -DskipTests

                    Write-Host ""
                    Write-Host "Build output:"
                    Get-ChildItem -Recurse -File -Include *.war, *.jar |
                        Where-Object { $_.FullName -notmatch "\\\\.git\\\\" } |
                        Select-Object -First 20 |
                        ForEach-Object { Write-Host $_.FullName }
                '''
            }
        }

        stage('Package Artifacts') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"
                    $ProgressPreference = "SilentlyContinue"

                    $env:VERACODE_API_KEY_ID = $env:VERACODE_API_ID
                    $env:VERACODE_API_KEY_SECRET = $env:VERACODE_API_KEY

                    $env:VERACODE_APP_NAME_RESOLVED | Out-File -FilePath app_name.txt -Encoding utf8

                    $sourceDir = if ($env:VERACODE_SOURCE_DIR) { $env:VERACODE_SOURCE_DIR } else { "app" }

                    Write-Host "Installing Veracode CLI..."

                    if (!(Test-Path "$env:USERPROFILE\\.veracode-cli\\veracode.exe")) {
                        & curl.exe -fsSL "https://tools.veracode.com/veracode-cli/install.ps1" -o "install.ps1"
                        powershell -NoProfile -ExecutionPolicy Bypass -File ".\\install.ps1"
                    } else {
                        Write-Host "Veracode CLI already exists at $env:USERPROFILE\\.veracode-cli\\veracode.exe"
                    }

                    $env:Path += ";$env:USERPROFILE\\.veracode-cli"

                    $veracodeExe = Join-Path $env:USERPROFILE ".veracode-cli\\veracode.exe"
                    if (!(Test-Path $veracodeExe)) {
                        $veracodeExe = "veracode"
                    }

                    & $veracodeExe version

                    Write-Host "Running Veracode autopackager on: $sourceDir"
                    New-Item -ItemType Directory -Force -Path verascan | Out-Null

                    & $veracodeExe package --source "$sourceDir" --output verascan --trust

                    Write-Host "Packaged artifacts:"
                    Get-ChildItem -Recurse verascan | Format-Table FullName, Length -AutoSize

                    $artifacts = Get-ChildItem -Path verascan -Recurse -File -Include *.war, *.jar, *.zip

                    if (!$artifacts -or $artifacts.Count -eq 0) {
                        Write-Error "No packaged artifacts found"
                    }

                    $workspace = (Get-Location).Path

                    $relativeArtifacts = $artifacts | ForEach-Object {
                        $_.FullName.Substring($workspace.Length + 1)
                    }

                    $relativeArtifacts | Out-File -FilePath artifact_list.txt -Encoding utf8

                    Write-Host "Artifacts found:"
                    Get-Content artifact_list.txt

                    Write-Host "Total artifacts: $($artifacts.Count)"
                '''
            }

            post {
                success {
                    stash name: 'verascan-bundle',
                          includes: 'verascan/**,artifact_list.txt,app_name.txt'

                    archiveArtifacts artifacts: 'artifact_list.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Veracode Security Scans') {
            parallel {

                stage('Agent-Based SCA') {
                    steps {
                        echo "Skipping Agent-Based SCA in this Windows-native version."
                        echo "The original SourceClear ci.sh script is Linux/macOS shell-based."
                    }
                }

                stage('Pipeline Scan (feature)') {
                    when {
                        allOf {
                            not { changeRequest() }
                            branch pattern: 'feature/*', comparator: 'GLOB'
                        }
                    }

                    steps {
                        unstash 'verascan-bundle'

                        powershell '''
                            $ErrorActionPreference = "Stop"
                            $ProgressPreference = "SilentlyContinue"

                            Write-Host "Downloading Veracode Pipeline Scanner..."
                            & curl.exe -fsSL "https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip" -o "pipeline-scan-LATEST.zip"

                            Expand-Archive -Path "pipeline-scan-LATEST.zip" -DestinationPath "." -Force

                            if (!(Test-Path "pipeline-scan.jar")) {
                                Write-Error "pipeline-scan.jar not found"
                            }

                            New-Item -ItemType Directory -Force -Path scan_results | Out-Null

                            $scanCount = 0
                            $artifacts = Get-Content artifact_list.txt

                            foreach ($artifact in $artifacts) {
                                $scanCount++
                                $artifactName = Split-Path $artifact -Leaf

                                Write-Host ""
                                Write-Host "=========================================="
                                Write-Host "Scanning [$scanCount]: $artifactName"
                                Write-Host "=========================================="

                                & java -jar pipeline-scan.jar `
                                    -f "$artifact" `
                                    -vid "$env:VERACODE_API_ID" `
                                    -vkey "$env:VERACODE_API_KEY" `
                                    --fail_on_severity "" `
                                    --issue_details true `
                                    -jo true

                                if ($LASTEXITCODE -ne 0) {
                                    Write-Host "Pipeline scan completed with findings for $artifactName"
                                }

                                if (Test-Path "results.json") {
                                    Move-Item -Force "results.json" "scan_results\\${artifactName}_results.json"
                                    Write-Host "Results saved: scan_results\\${artifactName}_results.json"
                                } else {
                                    Write-Host "WARNING: No results.json produced for $artifactName"
                                }
                            }

                            Write-Host ""
                            Write-Host "Pipeline Scan completed for $scanCount artifacts"
                        '''
                    }

                    post {
                        always {
                            archiveArtifacts artifacts: 'scan_results/**', allowEmptyArchive: true
                        }
                    }
                }

                stage('Pipeline Scan (PR Gate)') {
                    when {
                        changeRequest()
                    }

                    steps {
                        unstash 'verascan-bundle'

                        powershell '''
                            $ErrorActionPreference = "Stop"
                            $ProgressPreference = "SilentlyContinue"

                            Write-Host "Downloading Veracode Pipeline Scanner..."
                            & curl.exe -fsSL "https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip" -o "pipeline-scan-LATEST.zip"

                            Expand-Archive -Path "pipeline-scan-LATEST.zip" -DestinationPath "." -Force

                            if (!(Test-Path "pipeline-scan.jar")) {
                                Write-Error "pipeline-scan.jar not found"
                            }

                            New-Item -ItemType Directory -Force -Path scan_results | Out-Null

                            $failed = $false
                            $failedArtifacts = @()
                            $scanCount = 0
                            $artifacts = Get-Content artifact_list.txt

                            foreach ($artifact in $artifacts) {
                                $scanCount++
                                $artifactName = Split-Path $artifact -Leaf

                                Write-Host ""
                                Write-Host "=========================================="
                                Write-Host "Scanning [$scanCount]: $artifactName"
                                Write-Host "=========================================="

                                & java -jar pipeline-scan.jar `
                                    -f "$artifact" `
                                    -vid "$env:VERACODE_API_ID" `
                                    -vkey "$env:VERACODE_API_KEY" `
                                    --fail_on_severity "Very High, High" `
                                    --issue_details true `
                                    --policy_name "Veracode Recommended Very High" `
                                    -jo true

                                if ($LASTEXITCODE -ne 0) {
                                    $failed = $true
                                    $failedArtifacts += $artifactName
                                    Write-Host "WARNING: Policy gate failed for $artifactName"
                                }

                                if (Test-Path "results.json") {
                                    Move-Item -Force "results.json" "scan_results\\${artifactName}_results.json"
                                    Write-Host "Results saved: scan_results\\${artifactName}_results.json"
                                } else {
                                    Write-Host "WARNING: No results.json produced for $artifactName"
                                }
                            }

                            Write-Host ""
                            Write-Host "Pipeline Scan completed for $scanCount artifacts"

                            if ($failed) {
                                Write-Error "Policy gate failed for: $($failedArtifacts -join ', ')"
                            }
                        '''
                    }

                    post {
                        always {
                            archiveArtifacts artifacts: 'scan_results/**', allowEmptyArchive: true
                        }
                    }
                }

                stage('Sandbox Scan (release)') {
                    when {
                        allOf {
                            not { changeRequest() }
                            branch pattern: 'release/*', comparator: 'GLOB'
                        }
                    }

                    steps {
                        unstash 'verascan-bundle'

                        powershell '''
                            $ErrorActionPreference = "Stop"
                            $ProgressPreference = "SilentlyContinue"

                            Write-Host "Downloading Veracode Java API Wrapper..."
                            New-Item -ItemType Directory -Force -Path ".veracode" | Out-Null

                            $metadataUrl = "https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/maven-metadata.xml"
                            $metadataFile = ".veracode\\maven-metadata.xml"

                            & curl.exe -fsSL $metadataUrl -o $metadataFile

                            [xml]$metadata = Get-Content $metadataFile
                            $version = $metadata.metadata.versioning.latest

                            if (!$version) {
                                Write-Error "Failed to get API Wrapper version"
                            }

                            Write-Host "Downloading version: $version"

                            $wrapperUrl = "https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/$version/vosp-api-wrappers-java-$version-dist.zip"

                            & curl.exe -fsSL $wrapperUrl -o ".veracode\\dist.zip"

                            Expand-Archive -Path ".veracode\\dist.zip" -DestinationPath ".veracode" -Force

                            $jar = Get-ChildItem -Path ".veracode" -Recurse -File -Filter "VeracodeJavaAPI*.jar" | Select-Object -First 1

                            if (!$jar) {
                                Write-Error "VeracodeJavaAPI.jar not found"
                            }

                            Copy-Item -Force $jar.FullName "vosp-api-wrapper-java.jar"

                            $appName = Get-Content app_name.txt -Raw
                            $appName = $appName.Trim()

                            Write-Host "Uploading to Veracode Sandbox..."
                            Write-Host "Application: $appName"
                            Write-Host "Artifacts to upload:"
                            Get-ChildItem verascan

                            & java -jar vosp-api-wrapper-java.jar `
                                -vid "$env:VERACODE_API_ID" `
                                -vkey "$env:VERACODE_API_KEY" `
                                -action UploadAndScan `
                                -appname "$appName" `
                                -createprofile true `
                                -autoscan true `
                                -sandboxname "jenkins-release" `
                                -createsandbox true `
                                -filepath "verascan" `
                                -version "Release $env:BUILD_NUMBER"
                        '''
                    }
                }

                stage('Policy Scan (main)') {
                    when {
                        allOf {
                            not { changeRequest() }
                            branch 'main'
                        }
                    }

                    steps {
                        unstash 'verascan-bundle'

                        powershell '''
                            $ErrorActionPreference = "Stop"
                            $ProgressPreference = "SilentlyContinue"

                            Write-Host "Downloading Veracode Java API Wrapper..."
                            New-Item -ItemType Directory -Force -Path ".veracode" | Out-Null

                            $metadataUrl = "https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/maven-metadata.xml"
                            $metadataFile = ".veracode\\maven-metadata.xml"

                            & curl.exe -fsSL $metadataUrl -o $metadataFile

                            [xml]$metadata = Get-Content $metadataFile
                            $version = $metadata.metadata.versioning.latest

                            if (!$version) {
                                Write-Error "Failed to get API Wrapper version"
                            }

                            Write-Host "Downloading version: $version"

                            $wrapperUrl = "https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/$version/vosp-api-wrappers-java-$version-dist.zip"

                            & curl.exe -fsSL $wrapperUrl -o ".veracode\\dist.zip"

                            Expand-Archive -Path ".veracode\\dist.zip" -DestinationPath ".veracode" -Force

                            $jar = Get-ChildItem -Path ".veracode" -Recurse -File -Filter "VeracodeJavaAPI*.jar" | Select-Object -First 1

                            if (!$jar) {
                                Write-Error "VeracodeJavaAPI.jar not found"
                            }

                            Copy-Item -Force $jar.FullName "vosp-api-wrapper-java.jar"

                            $appName = Get-Content app_name.txt -Raw
                            $appName = $appName.Trim()

                            Write-Host "Uploading to Veracode Platform Policy Scan..."
                            Write-Host "Application: $appName"
                            Write-Host "Artifacts to upload:"
                            Get-ChildItem verascan

                            & java -jar vosp-api-wrapper-java.jar `
                                -vid "$env:VERACODE_API_ID" `
                                -vkey "$env:VERACODE_API_KEY" `
                                -action UploadAndScan `
                                -appname "$appName" `
                                -createprofile true `
                                -autoscan true `
                                -filepath "verascan" `
                                -version "main $env:BUILD_NUMBER"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Build finished with status: ${currentBuild.currentResult}"
            cleanWs(deleteDirs: true, notFailBuild: true)
        }

        failure {
            echo "Veracode pipeline failed. Review the stage logs and archived scan_results/."
        }
    }
}
