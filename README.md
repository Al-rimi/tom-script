# Tomcat Automation Script

This PowerShell script automates the process of building, deploying, and managing jsp web applications on Apache Tomcat, Opening browser, Reloading page with no user interaction.

## Features

<details>
<summary>Automatically builds projects using Maven</summary>

This feature ensures that the project is built using Maven, compiling the source code and generating a deployable WAR file.

```powershell
$process = Start-Process -FilePath "mvn" -ArgumentList "clean package" -PassThru -Wait -NoNewWindow

if ($process.ExitCode -ne 0) {
    Write-Host "[ERROR] Maven build failed. Exiting." -ForegroundColor Red | Out-Null
    exit 1
}
```
Explanation: The script starts the Maven process using `mvn clean package`. If the Maven build fails (i.e., the exit code is non-zero), an error message is displayed, and the script terminates.
</details>

<details>
<summary>Deploys WAR files to the Tomcat `webapps` directory</summary>

Once the Maven build completes successfully, the generated WAR file is deployed to the Tomcat `webapps` directory.

```powershell
$WAR_FILE = Get-ChildItem -Path "$PROJECT_DIR\target" -Filter "*.war" | Select-Object -First 1 -ExpandProperty FullName
$APP_NAME = [System.IO.Path]::GetFileNameWithoutExtension($WAR_FILE)

if (-not $WAR_FILE) {
    Write-Host "$ERROR No WAR file found. Closing Tomcat..." -ForegroundColor Red
    exit 1
}

Copy-Item -Path $WAR_FILE -Destination "$TOMCAT_HOME\webapps\"
INFO "New WAR file deployed"
```

Explanation: The script searches for the WAR file in the project's target directory and deploys it to the Tomcat `webapps` directory. If no WAR file is found, it terminates with an error message.
</details>

<details>
<summary>Starts or stops Tomcat based on its running status</summary>

The script includes functionality to start or stop Tomcat depending on the given action (`start` or `stop`).

```powershell
function Tomcat {
    param (
        [ValidateSet("start", "stop")] 
        [string]$Action
    )
    $javaExecutable = "$env:JAVA_HOME\bin\java.exe"

    Start-Process -FilePath $javaExecutable `
        -ArgumentList "-cp", $classpath, $catalinaOpts, $mainClass, $Action `
        -NoNewWindow `
        -RedirectStandardOutput $logOut `
        -RedirectStandardError $logErr `
        -Wait
}
```

Explanation: The `Tomcat` function takes an action parameter (`start` or `stop`) and starts or stops the Tomcat server accordingly by executing the Java process with the appropriate arguments. If Tomcat is not already running, it can be started; if it is running, it can be stopped.
</details>

<details>
<summary>Reloads the application if Tomcat is already running</summary>

If Tomcat is already running, the script will reload the application rather than restarting the server.

```powershell
if ($tomcatRunning) {
    $creds = New-Object System.Management.Automation.PSCredential("admin", (ConvertTo-SecureString "admin" -AsPlainText -Force))
    Invoke-WebRequest -Uri "http://localhost:8080/manager/text/reload?path=/$APP_NAME" -Method Get -Credential $creds | Out-Null
    INFO "Tomcat reloaded"
} else {
    Tomcat -Action start
}
```

Explanation: If Tomcat is running, the script uses the Tomcat manager's API to reload the application without restarting the server. If Tomcat is not running, it starts the server first.
</details>

<details>
<summary>Opens or refreshes the application in Google Chrome</summary>

The script ensures the application is opened or refreshed in Google Chrome after deployment.

```powershell
$chromeProcesses = Get-Process -Name "chrome" -ErrorAction SilentlyContinue

if ($chromeProcesses) {
    $chromeOpened = $false
    foreach ($process in $chromeProcesses) {
        $chromeTitle = $process.MainWindowTitle
        if ($chromeTitle -like "*$APP_NAME*") {
            $chromeOpened = $true
            [System.Windows.Forms.SendKeys]::SendWait("^{F5}") # Ctrl+F5 for hard refresh
            INFO "Google Chrome reloaded"
            break
        }
    }

    if (-not $chromeOpened) {
        INFO "Opening Google Chrome"
        Start-Process "chrome" "http://localhost:8080/$APP_NAME"
    }
} else {
    INFO "Opening Google Chrome"
    Start-Process "chrome" "http://localhost:8080/$APP_NAME"
}
```

Explanation: The script checks if Google Chrome is already open and refreshes the tab with the deployed application. If Chrome is not open, it launches a new instance with the application URL.
</details>

## Prerequisites
Before using this script, ensure the following requirements are met:

1. **[Java Development Kit (JDK)](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)**: Installed and `JAVA_HOME` environment variable is correctly set.
2. **[Apache Tomcat](https://tomcat.apache.org/download-90.cgi)**: Installed and `CATALINA_HOME` environment variable is correctly set.
3. **[Maven](https://maven.apache.org/download.cgi)**: Installed and added to the system's `PATH`.
4. **[Google Chrome](https://www.google.com/chrome/)**: Installed for automatic browser interaction.

### Windows PowerShell Execution Policy
By default, PowerShell may restrict running scripts due to security policies. To allow this script to run, execute the following command in PowerShell:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

This command grants permission to run locally created scripts.

## Installation
1. Copy the script to your desired directory.
2. Add the directory path to the system's **Environment Variables** to make the script accessible globally.
   - Open **System Properties** → **Environment Variables**.
   - Add the script's directory path to the **Path** variable under **System Variables**.

## Usage
To execute the script, simply run the `redeploy` script file. This must be executed in the directory containing the `pom.xml` file. It is crucial that the script is run from inside a **Maven project**.

Example:
```powershell
.\redeploy.ps1
```

This script assumes that the working directory contains the `pom.xml` file, indicating a valid Maven project. If the `pom.xml` is missing, the script will not run.

### Extendable Code
This script is designed to be **extendable**. You can add new functionality such as pre-deployment checks, custom notifications, or integrate with other tools.

### Add Custom Future Extensions:
- **Custom Deployment Notifications**: You can modify the script to send email or Slack notifications after deployment.
- **Custom Build Scripts**: Add new Maven goals or other build scripts as part of the `mvn clean package` command.

## VS Code Automatic Execution
For Visual Studio Code users, the script can be configured to run automatically on file save using the **Run on Save** extension.

1. Install the extension from this repository: [vscode-run-on-save](https://github.com/pucelle/vscode-run-on-save).
2. Add the following configuration to your VS Code settings:

```json
  "runOnSave.shell": "PowerShell",
  "runOnSave.commands": [
      {
          "match": ".*$",
          "command": "redeploy",
      }
  ],
  "runOnSave.defaultRunIn": "terminal",
```

## Error Handling
- If **JAVA_HOME** or **CATALINA_HOME** is not set correctly, the script will terminate with an appropriate error message.
- Maven build failures will stop further deployment.
- Missing WAR files will prompt Tomcat shutdown before exit.

## License
This script is licensed under the MIT [License](License).
