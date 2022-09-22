[![Build status](https://ci.appveyor.com/api/projects/status/ry28ehtnhrybtjti/branch/master?svg=true)](https://ci.appveyor.com/project/PowerShell/xwordpress/branch/master)

# xWordPress

The **xWordPress** module contains the **xWordPressSite** and **xIisWordPressSite** DSC resources that setup a WordPress Site.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Contributing

Please check out common DSC Resources [contributing guidelines](https://github.com/PowerShell/DscResource.Kit/blob/master/CONTRIBUTING.md).

## Resources

* **xIisWordPressSite** is a composite configuration that configures an IIS site with WordPress.
* **xWordPressSite** configures a WordPress Site.

### xIisWordPressSite

* **DestinationPath**: The path on the filesystem were the web site will be setup.
* **DownloadUri**: The URI to download the WordPress Zip file
* **PackageFolder**: The URI to download the WordPress Zip file computer name
* **PackageFolder**: The folder where the WordPress zip will be downloaded. This path MUST already exist.
* **Configuration**: The contents of the WordPress configuration file (wp-config.php).

### xWordPressSite

* **Uri**: The WordPress Site URI.
* **Title**: The WordPress Site Default page title.
* **AdministratorCredential**: The user name and password of the WordPress administrator to create when creating the site.
* **AdministratorEmail**: The email address of the WordPress administrator to create.
* **Ensure**: { Present }

## Versions

### Unreleased

* Updated sample to reference the renamed xPhpProvision resource in the xPhp
  module and version 5.7.15 of MySQL.
* Update appveyor.yml to use the default template.
* Added default template files .codecov.yml, .gitattributes, and .gitignore, and
  .vscode folder.
* Minor style changes to README.md.
* Added `Import-DscResource` to xIisWordPressSite to import dependent modules.

### 1.1.0.0

* Updated schema (ValueMap)
* Fixed encoding

### 1.0.0.0

* Initial release with the following resources
  * xWordPressSite
  * xIisWordPressSite

## Examples

### Setup a WordPress Site on a single node

This configuration will set a WordPress Site on a single node.
Note: This requires the following DSC modules:

* xPsDesiredStateConfiguration
* xMySql
* xPhp
* xWebAdministration

```powershell
# This configuration configures a Basic WordPress Site
# It requires xPhp, xMySql, xWordPress, and xWebAdministration
# Please review the note about the FQDN variable and
# about the URLs, they may need to be updated.
# ********* NOTE ***********
# If you are not targeting the local machine,
# or this does not resolve to the correct FQDN for the machine
# Update this to the FQDN of the target machine
# **************************
[string] $fqdn = [System.Net.Dns]::GetHostByName(($env:computerName)).HostName
Write-Host "Target Machine FQDN: $fqdn"
[string] $role = 'WordPress'
$dataRoot = Split-Path $MyInvocation.MyCommand.Path
$phpTemplatePath = join-Path $dataRoot "phpConfigTemplate.txt"
$WordPressTemplatePath = Join-Path $dataRoot "WordPressConfigurationTemplate.ps1"
if (-not (Test-Path $WordPressTemplatePath))
{
    $message = "Missing required file $WordPressTemplatePath"
    # This file is in the samples folder of the resource
    throw $message
}
$plainPassword = 'pass@word1'
$pwd = convertTo-SecureString -String $plainPassword -AsPlainText -Force
$WordPressUserName = 'WordPressUser'
$WordPressDatabase = 'WordPress'
# Generate the contents of the WordPress configuration
$wordPressConfig = & $WordPressTemplatePath -WordPressDatabase $WordPressDatabase -WordPressUserName $WordPressUserName -PlainPassword $plainPassword
# ********* NOTE ***********
# PHP and My SQL change their download URLs frequently.  Please verify the URLs.
# the WordPress and VC Redist URL change less frequently, but should still be verified.
# After verifying the download URLs for the products and update them appropriately.
# **************************
$configurationData = @{
    AllNodes = @(
        @{
            Role = $role
            NodeName = $fqdn
            PSDscAllowPlainTextPassword = $true;
            WordPress = @{
                Title = 'DSC WordPress Site Title'
                Admin = New-Object -TypeName System.Management.Automation.PSCredential -argumentlist ('DscAdmin', $pwd)
                Email = 'dscadmin@contoso.com'
                Uri = "http://$fqdn"
                DownloadURI = 'http://WordPress.org/latest.zip'
                Path = (Join-Path $env:SystemDrive  'wwwWordPress')
                Config = $wordPressConfig
                SiteName = 'WordPress'
                TemplatePath = $WordPressTemplatePath
                UserName = $WordPressUserName
                Database = $WordPressDatabase
                User = New-Object -TypeName System.Management.Automation.PSCredential -argumentlist ($WordPressUserName, $pwd)
            }

            Php = @{
                # Update with the latest "VC11 x64 Non Thread Safe" from http://windows.php.net/download/
                DownloadURI = 'http://windows.php.net/downloads/releases/php-5.5.14-nts-Win32-VC11-x64.zip'
                TemplatePath = $phpTemplatePath
                Path = "$env:SystemDrive\php"
                Vc2012RedistUri = 'http://download.microsoft.com/download/1/6/B/16B06F60-3B20-4FF2-B699-5E9B7962F9AE/VSU_4/vcredist_x64.exe'
            }
            PackageFolder = "$env:SystemDrive\packages"
            MySqlDownloadURI = 'http://dev.mysql.com/get/Downloads/MySQLInstaller/mysql-installer-community-5.7.15.0.msi'
            Credential = New-Object -TypeName System.Management.Automation.PSCredential -argumentlist ('userNameNotUsed', $pwd) #the password for root. no user name is needed as MySql installer is using only the user "root".
            PlainPassword = $plainPassword

         }
    )
}
# Configuration to configure a Single machine WordPress Site
Configuration WordPress
{
    # Import composite resources
    Import-DscResource -module xMySql
    Import-DscResource -module xPhp
    Import-DscResource -module xWordPress
    Node $AllNodes.NodeName
    {
        # Make sure we have a folder for the packages
        File PackagesFolder
        {
            DestinationPath = $Node.PackageFolder
            Type = 'Directory'
            Ensure = 'Present'
        }
        # Make sure PHP is installed in IIS
        xPhpProvision php
        {
            InstallMySqlExt = $true
            PackageFolder =  $Node.PackageFolder
            DownloadUri = $Node.Php.DownloadURI
            DestinationPath = $Node.Php.Path
            ConfigurationPath = $Node.Php.TemplatePath
            Vc2012RedistDownloadUri = $Node.Php.Vc2012RedistUri
        }
        # Make sure MySql is installed with a WordPress database
        xMySqlProvision mySql
        {
            DownloadURI = $Node.MySqlDownloadURI
            MySQLVersion = '5.7.15'
            RootCredential = $Node.Credential
            DatabaseName = 'WordPress'
            UserCredential =  $Node.WordPress.User
            PermissionType = 'ALL PRIVILEGES'
        }

        # Make sure the IIS site for WordPress is created
        # Note, you still need to create the actuall WordPress Site after this.
        xIisWordPressSite iisWordPressSite
        {
            DestinationPath = $Node.WordPress.Path
            DownloadUri = $Node.WordPress.DownloadURI
            PackageFolder = $Node.PackageFolder
            Configuration = $Node.WordPress.Config
        }
        # Make sure the WordPress site is present
        xWordPressSite WordPressSite
        {
            Uri = $Node.WordPress.Uri
            Title = $Node.WordPress.Title
            AdministratorCredential = $Node.WordPress.Admin
            AdministratorEmail = $Node.WordPress.Email
        }
        # Make sure LCM will reboot if needed
        LocalConfigurationManager
        {
            RebootNodeIfNeeded = $true
            # uncomment when debugging on WMF 5 and above.
            # DebugMode = $true
        }
    }
}
$outputFolder = "$env:USERPROFILE\Desktop\WordPress"
Wordpress -OutputPath $outputFolder -ConfigurationData $configurationData
Set-DscLocalConfigurationManager -path $outputFolder -verbose
Start-DscConfiguration -Wait -Verbose -path $outputFolder
```
