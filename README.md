# dotnet-remote-management-powershell-extension
Appdynamics .Net agent remote management powershell extension

# Overview

The AppDynamics PowerShell module for .NET agent management provides a set of cmdlets to administrator AppDynamics .NET agents. It includes commands to manage agent inventory, versioning, deployment, and configuration. The script uses PowerShell remoting to manage agents on remote servers so you can use the module to administer agents locally or access remote agents in a datacenter.


# Usage

1. Download the script.
2. Unblock the file in Windows.
3. Launch Power Shell and import the module: `Import-Module .\AppDynamics.psm1`
4. While loading, the module checks for PowerShell version 3.0 or later and if the current user has administrative permissions. It issues a warning if there are problems.
5. Execute commands.

# Requirements

- PowerShell 3.0 or later
- Windows 7/Windows Server 2003 or newer
- Administrator level access for local machine management
- Domain administrator access for remote agent management
- Enable PowerShell remoting for Windows remote management (enabled by default starting Windows 2008 R2).

# Common Command Options

## ComputerName

Specify a single remote host or a comma separated list of FQDN for remote hosts to execute the command, eg:

`ComputerName machine1.example.com, machine2.example.com, machine3.example.com`

Alternatively provide the list of servers in a text file in the format ( `Get-Content <path to text file>`), eg:

`ComputerName ( Get-Content .\Agents.txt )`

## RemoteShare

Change the location of shared directory on the remote machine where the script can copy installation files. The default value is `c$\Temp\AppDynamics\Install\`, eg:

## RestartIIS

Restart IIS after installation or configuration change.

## RestartWindowsServices

Specify a List of service names for services to restart, eg:

`RestartWindowsServices WindowsService1, WindowsService2, WindowsService3`

`RemoteShare d$\temp\AppDynamics\`

## Tier assignment

Remove the existing configuration before adding the new configuration.

### Windows services and standalone applications:

```
@{ Name=“<Windows service, or executable name>”; Tier=“<tier name>” }
```

### IIS Web Application
```
Site=“<site name>”; Path=“<virtual path>“; Tier=“<tier name>”
```

### Multiple tiers
```
@({ Name=“<IIS Application”; Path=“<virtual path>“; Tier=“<tier name>” }, { Name=“<Windows service, or executable name>”; Tier=“<tier name>” } )
```

## -RemotePath

Location of source install and configuration files, eg:

`-RemotePath d:\temp\AppDynamics\`

# Usage

## Get Agent Inventory

Get the agent version for every agent installed on machine

### Syntax
`Get-Agent`

### Examples
```
# Check agent inventory on the local machine.
Get-Agent

# Check agent inventory on multiple remote machines.
Get-Agent -ComputerName machine1.example.com, machine2.example.com, machine3.example.com

# Check agent inventory on a list of machines and export the results.
Get-Agent -ComputerName ( Get-Content .\servers.txt ) | Export-Csv .\Agents.txt
```

## Deploy Agents

The `Install-Agent` cmdlet installs new agents locally or on remote servers. Optionally restart IIS or Windows services after installation, supply a template configuration, or copy bits to the remote servers from a central location. Once install completes, `Install-Agent `automatically starts the `AppDynamics.Agent.Coordinator` service.

*See the bottom of this document for information on generating a template configuration file.*

### Syntax

`Install-Agent <path to MSI installer> <path to configuration template>`

### Examples

```
#Install the agent locally with no configuration file.
Install-Agent .\dotNetAgentSetup-x64-3.9.5.0.msi

#Install the agent locally with a configuration file.
Install-Agent .\dotNetAgentSetup-x64-3.9.5.0.msi .\template.xml

#Install the agent on a single remote host.
Install-Agent .\dotNetAgentSetup-x64-3.9.5.0.msi .\template.xml -ComputerName machine1.example.com

#Install the agent on multiple remote servers and restart IIS.
Install-Agent .\dotNetAgentSetup-x64-3.9.5.0.msi .\template.xml -ComputerName ( Get-Content .\servers.txt ) -RestartIIS

#Install the agent on multiple remote servers. Restart IIS and and a list of Windows services.
Install-Agent .\dotNetAgentSetup-x64-3.9.5.0.msi .\template.xml -ComputerName ( Get-Content .\servers.txt ) -RestartIIS -RestartWindowsServices Service1, Service2, Service3
```

## Deploy Agents Remote

When you install agents remotely, it is important to understand how the `Install-Agent` cmdlet copies the agent installation files to the remote machine(s):

1. When you begin the agent installer and template configuration are located on your local machine where your are running the PowerShell module or on a central shared repository location.
2. The `Install-Agent` cmdlet copies agent installation files to the default RemoteShare directory on the remote server(s): `c$\Temp\AppDynamics\Install\`
3. When the script executes on the remote machine, it looks for the installation files at the default `RemotePath` directory: `c:\Temp\AppDynamics\Install\`
4. Change the overrides using the `-RemoteShare` and `-RemotePath` command options.

### Example

```
#Install agents remotely. Restart IIS and a Windows Service. Change the location of the remote and local share paths.
Install-Agent .\dotNetAgentSetup-x64-3.9.5.0.msi .\template.xml -ComputerName ( Get-Content .\servers.txt ) -RestartIIS -RestartWindowsServices Service1 -RemoteShare d$\temp\AppDynamics\ -RemotePath d:\temp\AppDynamics\
```

## Upgrade Agents

You can use the `Install-Agent` cmdlet to upgrade the agent. When `Install-Agent` finds an existing agent on the server, it uses the following logic:

1. Get the local agent version.
2. Compare the installed agent version to the version from the `.msi` file.
3. If the installed version is the same or newer - do nothing. Otherwise continue.
4. If `-RestartIIS` or `-RestartWindowsServices` parameters were passed - stop IIS and/or stop windows service(s).
5. Uninstall existing agent.
6. Check the return code - if there is a problem, stop and report the error code. Otherwise continue.
7. Install new agent.
8. If `-RestartIIS` or `-RestartWindowsServices` parameters were passed - start IIS and/or start windows service(s).
9. Start `AppDynamics.Agent.Coordinator` service.

It is very important to restart IIS and/or Windows services instrumented with the .NET Agent. Otherwise uninstall could fail due to locked agent files.

## Modify Agent Configuration
The `Update-Configuration` cmdlet allows you to set the agent-controller connection. After you restart the instrumented application(s), either as part of the command or later, the new configuration takes effect and overrides any existing ones.

### Syntax

```
Update-Configuration -Host <FQDN of Controller> -Port <Controller port> -SSL -Application <Controller business application> -AccountName <Controller account> -AccessKey <Controller access key>
```


Examples

```
#Configure connection to a single-tenant controller with SSL. Restart IIS.
Update-Configuration -Host "controller.example.com" -Port 443 -SSL -Application "Demo Application" -ComputerName machine1.example.com -RestartIIS

#Configure connection to a multi-tenant controller without SSL. Restart IIS and Windows services.
Update-Configuration -Host "controller.saas.example.com" -Port 80 -SSL:$false -Application "Demo Application" -AccountName "MyAccount" -AccessKey "secret$" -ComputerName ( Get-Content .\servers.txt ) -RestartIIS -RestartWindowsServices Service1, Service2, Service3
 ```

*See .NET Agent Configuration Properties for definitions of the command options.*

## Modify Agent Configuration from Template
The `Update-ConfigurationFromTemplate` cmdlet applies a new configuration template to existing installed agents.

Once configuration is complete, `Update-Configuration` automatically restarts the `AppDynamics.Agent.Coordinator` service.

*See the bottom of this document for information on generating a template configuration file.*

### Syntax

```
Update-ConfigurationFromTemplate <path to configuration template>
```

### Examples

```
#Update agent config locally.
Update-ConfigurationFromTemplate .\template.xml

#Update agent configuration on a series of remote machines and restart IIS
Update-ConfigurationFromTemplate .\template.xml -ComputerName machine1.example.com, machine2.example.com, machine3.example.com

#Update agent configuration on a series of remote machines. Restart IIS and Windows Services
Update-ConfigurationFromTemplate .\template.xml -ComputerName ( Get-Content .\servers.txt ) -RestartIIS -RestartWindowsServices Service1, Service2, Service3

#Install agent configuration remotely. Restart IIS and a Windows Service. Change the location of the remote and local share paths.
Update-ConfigurationFromTemplate .\template.xml -ComputerName ( Get-Content .\servers.txt ) -RestartIIS -RestartWindowsServices Service1 -RemoteShare d$\temp\AppDynamics\ -RemotePath d:\temp\AppDynamics\
```

## Uninstall the Agent
The `Uninstall-Agent` cmdlet uninstalls the .NET Agent from local or remote server(s) from the command line.
It is important to restart the monitored applications and/or services during the restart.

### Syntax
```
Uninstall-Agent
```

### Examples
```
#Uninstall the agent locally. Restart IIS
Uninstall-Agent -RestartIIS

#Uninstall the agent locally. Restart IIS and Windows services.
Uninstall-Agent -RestartIIS -RestartWindowsServices Service1, Service2

##Uninstall the agent remotely. Restart IIS and Windows services.
Uninstall-Agent -RestartIIS -RestartWindowsServices Service1, Service2 -ComputerName machine1.example.com, machine2.example.com
```

## Restart the Agent Coordinator
`Restart-Coordinator` restarts `AppDynamics.Agent.Coordinator` windows service.

You don't have to use `Restart-Coordiinator` with the commands in this PowerShell module because the coordinator is internally restarted when required. This will be useful when you want to manually restart the Coordinator.

### Syntax
```
Restart-Coordinator 
```
### Examples
```
#Restart the agent coordinator on multiple remote machines.

Restart-Coordinator -ComputerName machine1.example.com, machine2.example.com
Restart-Coordinator -ComputerName ( Get-Content .\servers.txt )
```

## Read MSI File Version
The `Get-MsiVersion` cmdlet returns the version of a local msi installer package.

### Syntax
```
Get-MsiVersion <path to MSI installer>
```

## Instrument Standalone Applications
The `Add-StandaloneAppMontoring` cmdlet adds configuration for standalone applications to an existing configuration file. After it modifies the agent configuration, `Add-StandaloneAppMonitoring` restarts the `AppDynamics.Agent.Coordinator` service automatically

### Why restart IIS or a Windows service?
The most common use case is when those standalone application(s) are started from IIS or a windows service. In this case we need to restart parent service to start monitoring of the standalone app.You can use the `Add-StandaloneAppMonitoring` command programmatically. For example first, run a script that lists the executables to monitor and adds them to a data structure to pass to the `Add-StandaloneAppMapping` cmdlet.

### Syntax
```
Add-StandaloneAppMontoring @{ Name=“<executable name>”; Tier=“<tier name>” }
```
### Examples
```
#Instrument a standalone executable locally. Remove existing configurations.
Add-StandaloneAppMontoring @{ Name=“MyApp.exe”; Tier=“My App” } –Override
#Instrument a standalone executable on a remote machine.
Add-StandaloneAppMontoring @( @{ Name=“MyApp1.exe”; Tier=“My App1” }, @{ Name=“MyApp2.exe”; Tier=“My App2” } ) -ComputerName machine1.example.com
#Instrument multiple standalone executables locally. Restart IIS and a Windows service. Remove existing configurations.
Add-StandaloneAppMontoring @( @{ Name=“MyApp1.exe”; Tier=“My App1” }, @{ Name=“MyApp2.exe”; Tier=“My App2” } ) -RestartIIS -RestartWindowsServices svc1 -Override
```

## Instrument Windows Services
The `Add-WindowsServiceMontoring` cmdlet adds configuration for Windows services to an existing configuration file. After it modifies the agent configuration, Add-WindowsServiceMonitoring restarts the `AppDynamics.Agent.Coordinator` service automatically

### Syntax
```
Add-WindowsServiceMontoring @{ Name=“<executable name>”; Tier=“<tier name>” }
```

### Examples
```
#Instrument a Windows service locally. Remove existing configurations.
Add-WindowsServiceMonitoring @{ Name=“MySvc”; Tier=“My Svc” } –Override
#Instrument two Windows Services remotely.
Add-WindowsServiceMonitoring @( @{ Name=“MySvc1”; Tier=“My Svc1” }, @{ Name=“MySvc2”; Tier=“My Svc2” } ) -ComputerName machine1.example.com
#Instrument two Windows services locally. Restart IIS. Restart the 
Windows services.
Add-WindowsServiceMonitoring @( @{ Name=“MySvc1”; Tier=“My Svc1” }, @{ Name=“MySvc2”; Tier=“My Svc2” } ) -RestartIIS -RestartWindowsServices MySvc1, MySvc
```

The following example automation script discovers local windows services which have names starting with “MyCompany” and adds those for monitoring
```
$services = @() 
foreach($i in (Get-Service -Name “MyCompany.*" })) 
{ 
	$svc = @{ Name=$i.Name ; Tier=$i.Name } 
	$services += $svc 
} 
Add-WindowsServiceMonitoring $services -RestartWindowsServices ( $services | { Select $_.Name} )
```

## Instrument IIS web applications
The `Add-IISApplicationMontoring` cmdlet adds configuration for IIS applications to an existing configuration file. After it modifies the agent configuration, `Add-WindowsServiceMonitoring` restarts the `AppDynamics.Agent.Coordinator` service automatically.

### Syntax
```
Add-IISApplicationMontoring @{ Site=“<site name>”; Path=“<virtual path>“; Tier=“<tier name>” }
```

### Examples
```
#Instrument the the Default Web Site locally.
Add-IISApplicationMonitoring @{ Site=“Default Web Site”; Path=“/“; Tier=“Web” }
#Instrument the Default Web Site locally. Remove existing configurations.
Add-IISApplicationMonitoring @{ Site=“Default Web Site”; Path=“/“; Tier=“Web” } -Override 
#Instrument the Default Web Site locally. Restart IIS
Add-IISApplicationMonitoring @{ Site=“Default Web Site”; Path=“/“; Tier=“Web” } -RestartIIS
#Instrument the Default Web Site remotely. 
Restart IIS Add-IISApplicationMonitoring @{ Site=“Default Web Site”; Path=“/“; Tier=“Web” } -ComputerName machine1.example.com -RestartIIS
```

The following automation script example scan local IIS web sites and instruments the sites ending with ".com" for monitoring:

```
$apps= @() 
foreach($i in (Get-WebSite -Name “*.com" })) 
{ 
	$app= @{ Site=$i.Name; Path=“/“; Tier=$i.Name } 
	$apps += $app 
} 
Add-IISApplicationMonitoring $apps -RestartIIS
```

## Scripting Agent Configuration Using PowerShell Script Block
The `Update-ConfigirationFromScript` cmdlet is a sophisticated automation example. It lets you use a script block to define monitoring configurations individually for each server.This script block uses smart logic that determines which applications to monitor and configures them accordingly.

### Syntax
```
Update-ConfigurationFromScript { <discoveryfunction> }
```
***Note*** `discoveryfunction` *is a user-defined method. It can execute locally or remotely.*

### Examples
```
#Update configurations locally. Remove existing configurations. 
Update-ConfigurationFromScript { MyDiscoveryFunction } –Override
#Update configurations locally. Restart IIS and Windows services. 
Update-ConfigurationFromScript { MyDiscoveryFunction } -RestartIIS -RestartWindowsServices Service1, Service2
#Update configurations remotely. Restart IIS and Windows services. Remove existing configurations 
Update-ConfigurationFromScript { MyDiscoveryFunction } -RestartIIS -RestartWindowsServices svc1, svc2 -ComputerName ( Get-Content .\servers.txt ) -Override
```

This example function should return a hashtable with optional properties specifying which standalone application(s), windows service(s) and/or IIS application(s) to monitor

```
function MyDiscoveryFunction() 
{ 
	# Package return into a hashtable 
	[hashtable]$configuration = @{} 
	# IIS array allows to monitor IIS applications 
	$configuration.IIS = @() 
	$configuration.IIS += @{ Site=“Default Web Site”; Path=“/“; 
	Tier=“Web” } 
	# Standalone allow to monitor standalone applications 
	$configuration.Standalone = @() 
	$configuration.Standalone += @{ Name=“MyApp.exe”; Tier=“My App” } 
	# WindowsService is an array with the list of windows service to be 
	monitored 
	$configuration.WindowsService = @() 
	$configuration.WindowsService += @{ Name=“MySvc”; Tier=“My Svc” } 
	return $configuration 
}
```

This scenario enables dynamic discovery that can run on each remote server and return precise configuration.

Below is another example script that checks if the IIS web server is installed and then enables for monitoring only sites ending with “.com”. Additionally, it discovers local windows services with the “MyCompany” in the name
```
function MyDiscoveryFunction() 
{ 
	# Package return into a hashtable 
	[hashtable]$configuration = @{} 
	
	# Check if IIS is installed 
	$w3svc = Get-Service W3SVC -ErrorAction SilentlyContinue 
	if($w3svc -ne $null) 
	{ 
		$websites = Get-WebSite -Name “*.com" 
	}

	if($websites.Count -gt 0) 
	{ 
		$configuration.IIS = @() 
		foreach($website in $websites) 
		{ 
			$configuration.IIS += @{ Site = $website.Name; Path = "/"; Tier 
			= $website.Name } 
		} 
	} 
 
	# Check for windows services with naming starting "MyCompany." 
	if($services.Count -gt 0) 
	{ 
		$services = Get-Service -Name “MyCompany.*" 
	} 
	$configuration.WindowsService = @() 
	foreach($service in $services) 
	{ 
		$configuration.WindowsService += @{ Name=$service.Name ; Tier=$service.Name } 
	} 
	 
	# Return resulted configuration 
	return $configuration 
}
```

# Uninstalling/upgrading agent problems - Troubleshooting Tips

The most common issue with uninstallation/upgrade is caused by monitored processes locking the agent files. When this happens PowerShell commands return only the generic messages from the Windows installer. For example: “Error 1603”.

How to approach this situation:
1. Access the target server.
2. Open the event viewer (eventvwr).
3. Examine events in the “Application” log around the time of the installation failure.
4. There should be multiple entries from the “MsiInstaller” source.
5. Find the one with which have error details.

Additionally grab the MSI log from the server, logs are `%TEMP\AppDynamicsInstall.log` and `%TEMP\AppDynamicsUninstall.log`
If you can’t identify the root cause and resolve the problem then contact AppDynamics support.
