# LTGReport

Currently, this script is designed solely to get Data from various sources and push that data to an Azure SQL database used for Tableau. It operates by looping through the directories in the Plugins directory. It calls each script in each Plugin directory in alphabetical order.  Each script is dot-sourced by the main script (LTGReport.ps1), so the order in which the plugin scripts operate is important.

The main script has been written to allow for different types of output, eventually it should replace Veeam/vCheck reports and even be able to upload data to ConnectWise, IT Glue, etc.

## Tableau Setup

Before a new organization can be brought into the Tableau database, you must complete some initial config in the Tableau Database. The following commands can be used to do this. All of the following commands rely on the two modules mentioned above, you must also have an open connection to the Tableau Database as an administrative user.

```powershell
# Import Modules
Import-Module SqlTools
Import-Module Tableau

# Setup Connection Parameters
$SqlParameters = @{}
$SqlParamters.Credential       = Get-Credential # give your admin creds when prompted
$SqlConnectParameters.Server   = 'ltgsql.database.windows.net'
$SqlConnectParameters.Database = 'ltgba'

# Connect to database
$SqlConnect = Connect-SqlServer @SqlConnectParameters
```

### Add New User to Tableau

1. Decide on a new username, all customer usernames should start with `cust_`
1. Generate a random password with no special characters and save to to lastpass with the following naming convention.

    `LTG - Tableau DB User - cust_<customer>`

1. Add the user to Tableau with the following command, you will be prompted for the desired username and password.

    ```powershell
    New-TbUser
    ```

### Add Organization to Tableau

1. Add new Organizations to Tableau with the following command. This will return an OrgId that you will need for the next step.

    ```powershell
    New-TbOrganization -OrgName 'My New Customer' -OrgCode 'MNC'
    ```

### Add Sites to Tableau

1. Each site that you intend to pull data for, will need to be added to Tableau. This requires the OrgId that's returned from `New-TbOrganization`. You can also retrieve these with `Get-TbOrganization`. The following example is to add the Lockstep Kennesaw office to Tableau. This will return a SiteId needed for the next step.

    ``` powershell
    $SiteParams = @{}
    $SiteParams.SiteName  = 'Kennesaw Office'
    $SiteParams.SiteCode  = 'WST'
    $SiteParams.OrgId     = '103'
    $SiteParams.Street    = '1350 Wooten Lake Rd'
    $SiteParams.Suite     = '302'
    $SiteParams.City      = 'Kennesaw'
    $SiteParams.State     = 'GA'
    $SiteParams.ZipCode   = '30144'
    New-TbSite @SiteParams
    ```

### Add Assets to Tableau

1. All data gathered for Tableau is gathered about "Assets". These can be Virtual Machines, Veeam Servers, Firewalls, etc. Before you can upload data about an Asset, it needs to be created in the database. You can do so with the following command. This will return an Asset Id which is need to configure the data gathering script (LTGReport.ps1).

    ```powershell
    $AssetParams = @{}
    $AssetParams.Type        = 'Veeam Server'
    $AssetParams.Hostname    = 'ltg-wst-bkp-001'
    $AssetParams.Fqdn        = 'ltg-wst-bkp-001.west.side'
    $AssetParams.Description = 'Kennesaw Veeam Server'
    $AssetParams.OrgId       = '111'
    $AssetParams.SiteId      = '111'
    New-TbAsset @AssetParams
    ```

### Add Public IP to Tableau

1. The public IP of the server(s) running the LTGReport script needs to have access to the Azure Sql Database used for Tableau. This must be added from somewhere that already has access to the Database. You must be connected to the 'master' database to run the following commands.

    ```powershell
    # Setup Connection Parameters
    $SqlParameters = @{}
    $SqlParamters.Credential       = Get-Credential # give your admin creds when prompted
    $SqlConnectParameters.Server   = 'ltgsql.database.windows.net'
    $SqlConnectParameters.Database = 'ltgba'

    # Connect to database
    $SqlConnect = Connect-SqlServer @SqlConnectParameters

    # Add new IP to database
    New-TbPublicIp -IpAddress '1.1.1.1' -Description 'Customer - Server Running Script - Site Name'
    ```

## Install the Report

### Pre-Reqs

* Powershell 5+ (works with core, but all plugins may not)
* [Tableau PowerShell Module](https://github.com/LockstepGroup/Tableau) (Private Repo, requires github credentials)

    ```powershell
    Get-Stage Tableau
    ```

* [SqlTools PowerShell Module](https://github.com/LockstepGroup/SqlTools)

    ```powershell
    Install-Module SqlTools
    ```

### Installation

You can get this straight from github or with Strap.  Either way, it's a private repo and requires credentials.

```powershell
Get-Stage LTGReport
```

By default, the script will be installed to C:\Lockstep\LTGReport. Next you will need to configure the script. There are config.json.example files in the root directory and each Plugin directory. Copy these to just plain config.json and fill them out accordingly. Passwords need to stored as a SecureString encrypted with an AES key that is stored in the main config.json file (Eventually I'll write cmdlets to make this all easier hopefully). Instructions for generating an AES key and using it to create SecureStrings is further down.

Since the main config.json file contains the encryption key for passwords, this file should be secured with NTFS permissions at minimum.

### Usage

Once you've got all the config files setup you can use the script in two ways.

* Manually - Best for testing new plugins or initial setup. Several parameters are available for various situations, you can use Get-Help to see them. The Example below is dot-sourced because I assume you're troubleshooting.

    ```powershell
    cd c:\Lockstep\LTGReport
    . ./LTGReport.ps1
    ```

* Scheduled Task - This is the primary way this should be used for permanent installation. You can use the provided `install.ps1` script to register a scheduled task with the following settings.
  * Action: Start a program
  * Program/Script: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`
  * Arguments: `-ExecutionPolicy Bypass -Command "& C:\Lockstep\LTGReport\LtgReport.ps1 -Verbosity 2"` (Verbosity is optional, but this creates a log file on each run)
  * UserName/Pass: the script will prompt you for creds

## Configuration Files

Before Running the report you'll need to configure it. There are Config files for each plugin and a global config. These are JSON files so they're easy to edit. Please see the note [below](#storing-passwords-securely) for storing password securely in these files.

### Global Config

The config.json file in the root folder of the script is the Global Config file and contains the following sections.

#### SqlConnection

Contains the detail of the Sql Server to send data to.

| Key | Type | Value |
| --- | ---- | ----- |
| Server | String | fqdn of sql server |
| Database | String | name of the database to store data to |
| Username | String | username with auth to write to database |
| Password | SecureString | password for database auth |

#### Key

Contains key for aes encryption of SecureStrings.

| Key | Type | Value |
| --- | ---- | ----- |
| Key | Array | AES key in byte array form |

### Plugin Config

Config files for plugins will have varying data. The following sections are required.

#### Global

Contains configuration data to be used with each script in the Plugin directory. Each Plugin directory should contain it's own README.md. Requires the following information at minimum.

| Key | Type | Value |
| --- | ---- | ----- |
| Enabled | bool | One of the criteria that is checked to see whether or not to execute the Plugin. |

#### Device

Contains an array of devices to loop through for the plugin. Each Device requires the following information at minimum.

| Key | Type | Value |
| --- | ---- | ----- |
| AssetId | int | Tableau Asset Id |

### Storing Passwords Securely

In order to store passwords, or any sensitive information, securely in the config file, we take advantage of SecureStrings.  All SecureStrings should be converted to plain text using an AES Encryption Key which is stored in the Global config.json file under the `Key` node.  Here are the steps to make that happen.

1. Create an AES Key

    ```powershell
    Import-Module c:\Lockstep\LtgReport\Module\LtgReport
    $Key = New-LtgEncryptionKey
    $Config = Import-LtgConfigFile c:\Lockstep\LtgReport\config.json
    $Config.Key = $Key
    $Config | ConvertTo-Json | Out-File c:\Lockstep\LtgReport\config.json
    ```

2. Encrypt a string
    ```powershell
    $Config = Import-LtgConfigFile c:\Lockstep\LtgReport\config.json
    $EncryptedString = New-LtgEncryptedString -PlainTextString 'My Super Secret Password' -AesKey $Config.Key
    ```