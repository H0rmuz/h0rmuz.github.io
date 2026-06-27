---
layout: post
title: 'Assume Breach Adversary Simulation Approach'
tags:
 - adversarial Simulation
 - c2
hero: https://plus.unsplash.com/premium_photo-1673543763969-1d54002352d0?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1470&q=80
overlay: red
---

{: .lead} <!–-break–>

## Pre-Engagement Setup

### VMs

- **Ubuntu Development:** Ubuntu 20.04 has a known compatibility issue with DonPapi that requires installing Python 3.11 (the latest version).
- **Kali Linux:** Deployed strictly for operational comfort and accessibility.
- **Win11 Development:** Configured for exploit compilation and payload testing.
- **Custom Win11:** Dedicated to maintaining a steady connection to our internal company VPN without disconnecting the physical host machine from the customer's target VPN infrastructure.

### Notes

- **OneNote or Obsidian Calendar Section:** Document everything you do on an everyday basis. Maintain a precise timeline tracking your exact actions, timestamps, and command outputs.
- **Operational Section:** Dedicate entirely new pages to track active attack vectors, specific phases, and testing modules.
- **Storyline Section:** Organize your findings chronologically day-by-day so the data is immediately structured and ready for reporting.
- **Enumeration Section:** Store all raw ADExplorer files, LDAP query outputs, server lists, and infrastructure data. Your primary goal is to map out the entire domain architecture.

## Assume Breach Scenario

The customer establishes the engagement parameters by providing two distinct access vectors:

1. **RDP Access:** Access is granted to a low-privilege workstation cloned from a real, newly onboarded employee account to closely simulate an insider threat scenario.
2. **VPN Access:** Access is provided to simulate an Advanced Persistent Threat (APT) actor that has purchased valid access vectors or harvested VPN credentials via targeted phishing campaigns.

## Timeline

- **First 10 Days:** Move deliberately and slowly to avoid triggering proactive SOC detections. Focus purely on low-impact enumeration via targeted LDAP queries and silent network share browsing.
- **Beacon Catching:** Establish initial footholds quietly during the opening days. During the final days of the assessment, purposely escalate your operational noise level to actively stress-test the defensive detection and response capabilities of the SOC.

## 1. RDP Access

When accessing the environment via RDP, prioritize conducting actions through graphical user interface (GUI) applications as much as possible to minimize the volume of generated command-line telemetry. Because command-line interfaces (CLI) are aggressively logged and monitored by modern security solutions, execute CLI actions with extreme caution.

The primary objective is starting the initial reconnaissance phase to map out how to abuse the environment, ultimately preparing to drop a persistence beacon while masking its presence using solid OPSEC. Your primary focus must be credential hunting; uncovering valid credentials is the only reliable way to execute lateral movement.

Never run high-telemetry commands like `whoami /all` directly on the host. If you need to collect equivalent user token information, use an in-memory Beacon Object File (BoF) or compile a custom execution binary (`.exe`) that requests this data directly via native Windows APIs that query your token context. Additionally, keep credential harvesting top-of-mind at all times; systematically audit every accessible, high-interest flat text file for sensitive strings.

### Host Recon Methodology

- **RDP Information:** Gather all connection and interface data.
- **Internet Connectivity:** Verify outward connectivity directly from the GUI, and check the current local IP address configuration.
- **DNS Configuration:** Identify assigned DNS servers. In the vast majority of enterprise environments, the primary DNS servers will coincide directly with the Domain Controllers (DCs).
- **Subnet Mapping:** Check local subnet masks and adjacent network ranges.
- **Hostname Verification:** Query the current hostname via the GUI (System Information). Ensure you change the hostname of your attack platform and the provided RDP machine to an unassigned name that mirrors the target company's exact naming convention. This prevents detections triggered by default hostnames like `KALI` or `WINDOWS`, which SOC teams actively monitor and alert on.
- **Environment Variables:** Inspect current environment variables set via `cmd.exe`. Check the system `PATH` thoroughly to identify potential local Privilege Escalation (PE) vectors, such as DLL hijacking vulnerabilities.
- **User Domain:** Identify the exact assigned domain name via `USERDOMAIN`.
- **Logon Server:** Determine the active authenticating Domain Controller via `LOGONSERVER`.
- **Username:** Verify the active executing context via `USERNAME`.
- **Installed Applications:** Audit installed software within both the `C:\Program Files` and `C:\Program Files (x86)` directories to identify potential local privilege escalation opportunities.
- **OPSEC Staging Paths:** Analyze the file system to determine where you can safely conceal your operational beacons. Ideal locations include native application pathways, such as specific PowerShell folders or deep inside hardware driver directories (e.g., Realtek driver paths).
- **AV/EDR Detection:** Identify the active security solutions defending the host (e.g., Windows Defender, SentinelOne, CrowdStrike).
- **Password Managers:** Search for default password management applications. For example, if you obtain local administrative privileges on a machine running KeePass, you can modify the application's XML configuration file to insert a custom payload synchronization hook. This will transparently exfiltrate database credentials in cleartext to a remote platform (ensure you utilize the same compromised staging machine to handle this traffic safely).
- **ProgramData Directory:** Audit `C:\ProgramData` for the same operational reasons. This directory is optimal for inspecting application logs and often features permissive read/write permissions for standard user groups, making it a reliable directory to stage malicious binaries. Review application log files to uncover hidden system behavior.
- **Running Processes:** Use Task Manager to comprehensively audit currently executing processes.
- **Credential Guard Verification:** Check the process list for the presence of `LsaIso.exe` (Credential Guard). If this process is active, dumping the LSASS memory space will yield encrypted hashes that cannot be directly utilized. Additionally, verify if LAPS is implemented across the environment; when LAPS is active, local administrator credentials change automatically, though misconfigured service account properties may occasionally leak cleartext passwords within accessible attributes.
- **Process Analysis:** Continuously analyze running processes to identify legitimate software footprints that can be used to seamlessly mask and blend your beacon's execution.
- **Running Services:** Use Task Manager to map all active system services and understand what background components are executing.
- **Service Mapping:** Analyze running services to determine optimal native service names to mimic when configuring your persistent beacon architecture.
- **Remote Network Resolution:** Perform targeted `nslookup` queries against the remote IP addresses discovered during your initial enumeration.
- **Firewall Status:** Check the firewall configuration via the Control Panel. If the firewall is disabled, you can freely leverage RPC services to execute lateral movement. Long story short: an unmanaged firewall allows you to leverage remote RPC interfaces to remotely instantiate and execute new system services.
- **Inbound RPC Rules:** Inspect inbound firewall definitions to verify if RPC and RPC EPMAP rules (the four core rules) are enabled. If active, they provide a reliable channel for lateral movement.
- **User Profiles:** Audit the `C:\Users` directory to identify which specific corporate users have previously logged into the machine.
    

## Active Directory Recon Methodology via ADExplorer

Keep in mind that expanding a node (clicking a `+` icon) within Active Directory Explorer triggers live, active LDAP queries across the domain; move slowly, deliberately, and quietly.

Never generate full active directory snapshots inside the target network; they utilize extensive wildcard filters (`*`) that create massive, highly visible traffic anomalies that easily trigger SOC alarms. Instead, use snapshots locally if you can safely extract an initial database instance, perform your AD modifications, and retake the snapshot offline to delta the structural changes. This approach allows you to analyze AD information directly without touching or alerting the live target environment.

High-frequency queries and broad wildcard (`*`) searches are quickly detected by modern security engineering teams. In contrast, highly targeted, precise LDAP queries rarely trigger alerts because attempting to monitor every single legitimate LDAP query within an enterprise network would generate an overwhelming volume of false positives for the blue team.

- **Execution Path:** Run ADExplorer directly via its Sysinternals WebDAV path: `\\live.sysinternals.com\tools\ADExplorer64.exe`. This method runs the utility directly from memory without writing the executable binary to the local host disk.
- **Authentication:** Establish a connection to the Domain Controller using your valid domain user credentials.
- **Configuration Audit:** Review the configuration partition to map out the organization's network sites and analyze how its infrastructure is geographically distributed.
- **Sites and Services:** Highlighting a specific network subnet within the interface allows you to parse the `siteObject` property to extract details about its corresponding geographic location.
- **Domain Settings and Account Quotas:** Review the domain settings to pull the active password policy parameters (including minimum/maximum password length, lockout duration, and the lockout threshold). Additionally, audit the `ms-DS-MachineAccountQuota` attribute. By default, Active Directory permits any authenticated domain account to join up to 10 machine accounts to the domain. While this capability can be restricted via Group Policies, always verify this value; if it is set to `0`, standard accounts are blocked from adding machines. Concurrently, inspect sensitive attributes like `userPassword`, `unicodePwd`, `unixUserPassword`, `os400Password`, and `msSFU30Password`. These properties can occasionally contain cleartext passwords stored as decimal ASCII arrays. If discovered, extract these arrays and use CyberChef (applying a 'From Decimal' recipe) or an offline string tool to convert the values back into cleartext strings.
- **Trusts and Delegations:** Identify active domain trusts by searching for objects possessing an `objectClass` attribute set directly to `trustedDomain`. Thoroughly audit critical trust metadata, including `trustType` and `trustDirection`.
- **Navigation Tip:** ADExplorer will list out all discovered trusts. Double-clicking any specific result entry in the lower panel of the interface automatically jumps your view to that object's exact hierarchical location in the Active Directory tree, allowing you to easily audit properties like `trustType` and `trustDirection`.
- **Server Inventory:** Expand the primary domain object to inspect its layout. Execute a targeted right-click search against the `operatingSystem` attribute, filtering for strings that contain the word `erve` (or similar partial strings) to isolate all domain servers.
- **Legacy OS Auditing:** Identify legacy or end-of-life operating systems. For example, if your enumeration uncovers a host vulnerable to an exploit like EternalBlue, document the finding, notify the customer, and ask if they maintain an isolated testing environment to safely demonstrate the vulnerability. Never run unreliable or unstable exploit payloads against production infrastructure to avoid causing a catastrophic Denial of Service (DoS) condition.
- **LAPS Attribute Inspection:** Check for Local Administrator Password Solution (LAPS) implementations by inspecting the `ms-Mcs-AdmPwd` attribute. If LAPS is improperly secured or misconfigured, the local administrator credentials will be visible in plaintext directly within this field.
- **Administrative Count Tracking:** Audit objects where the `adminCount` attribute is set explicitly to `1`. This value indicates that the account is currently or was previously a member of a protected administrative group within Active Directory (such as Domain Admins or Account Operators).
- **Description Field Scraping:** Execute a targeted search filtering for any objects where the `description` attribute is not empty, then manually analyze the text strings to uncover misplaced credentials or sensitive infrastructure details.
- **msExchRequireAuthToSendTo Auditing:** Inspect the `msExchRequireAuthToSendTo` attribute. If you need to deliver highly targeted phishing payloads to a specific internal distribution group from an external address, query AD to find distribution lists that accept external mail streams. The `msExchRequireAuthToSendTo` attribute identifies these boundaries; when it evaluates to `False`, any external unauthenticated sender can successfully broadcast mail to the entire group. You can double-click any distribution group within the search results to inspect its `member` attribute and extract the full recipient roster. While pulling individual addresses manually is tedious, broadcasting your pretext directly to an established group alias is fast and carries significantly higher authenticity for the recipients.
- **OPSEC Search Protocol:** Broad wildcard searches are extremely loud. Avoid using `*` globally; instead, execute highly specific searches or systematically query structural names character-by-character (e.g., indexing sequentially by a, b, c...) to blend in with legitimate administrative queries.
### Other Attributes with Potential Passwords

- `userPassword`
- `unicodePwd`
- `unixUserPassword`
- `msSFU30Password`
- `os400Password`

If you uncover credentials populated within these attributes, they are frequently stored as decimal ASCII arrays. Use an offline or secure local converter (such as a secure implementation of an ASCII-to-string utility) to reconstruct the plaintext password string. Exercise extreme caution and never input sensitive production credentials into untrusted public online tools.

## Credentials Hunting on Enumerated Shares

If you discover an interesting network share, review its access control properties before attempting to browse it. Verify which specific domain groups hold read/write privileges and ensure your current user context belongs to those authorized groups.

Maintain a rigorous log within the calendar/timeline section of your notes documenting every enumerated network share and its corresponding findings. For high-value shares containing critical data, dedicate a specific project page and organize clear screenshots of the file structures.

When accessing flat files like `.txt`, `.pdf`, or image formats, you can open them directly. However, for structured files like `.xlsx` or `.docx`, inspect their properties and metadata before opening them to ensure the application launch does not trigger host-based security alerts or telemetry. If valuable data is identified, take a local screenshot or copy the raw text directly into your notes (Note: extract the raw content string only; do not copy or transfer the actual file itself to avoid creating file-system movement artifacts).

If new credentials are recovered, immediately duplicate your reconnaissance process using tools like `runas` to identify newly accessible network shares and directories.

### Credential Harvesting Workflow

- Analyze all available network shares, flat text files, and high-value data repositories.
- Systematically check valid credential permissions using direct server paths: `\\servername`.
- If valid credentials are recovered, proceed immediately to the Lateral Movement section.
- **Last Resort PC Check:** If no data is uncovered across the network, check the local system's Explorer path history by typing `\\` directly into the run dialog or address bar, and analyze the cached autocompletion paths (Note: this is a higher-risk operational action).

## 2. VPN Access

When operating from a VPN access point, the core scanning methodology, technical objectives, and post-exploitation constraints remain identical to those executed during RDP access.

### Active Directory Recon Methodology via LDAPSearch

Leverage the following references to structure precise LDAP filtering syntax:

- `https://gist.github.com/jonlabelle/0f8ec20c2474084325a89bc5362008a7`
- `https://learn.microsoft.com/en-us/archive/technet-wiki/5392.active-directory-ldap-syntax-filters`
- `https://jonlabelle.com/snippets/view/markdown/ldap-search-filter-cheatsheet`

Using `ldapsearch`, you can construct highly granular queries. Ensure all queries remain strictly targeted to avoid triggering heuristic network detections or raising suspicion among the blue team.

To convert raw output strings and dates into readable formats, use the offline custom version of following utilities:

- **User Account Control Decoding:** `https://www.techjutsu.ca/uac-decoder`
- **LDAP Timestamp Conversion:** `https://www.epochconverter.com/ldap`
    

Always change your local Kali Linux/Ubuntu machine provided hostname before executing `ldapsearch` commands from your attack platform. Ensure your hostname matches the naming convention of the target company's network or precisely mirrors the RDP host provided to you, mimicking legitimate internal assets to avoid raising alarms in the SIEM.

To extract a comprehensive server list (remember that with active ADExplorer views, exporting requires a snapshot and the GUI caps out at 1,000 visible entries), use the following structured queries:


```
# 2. Search the entire directory for computer objects filtering for names or operating systems containing 'erve' (e.g., Server)
ldapsearch -x -LLL -H ldap://yourdomain.com -D "user@yourdomain.com" -w "password" "(&(objectCategory=computer)(|(name=*erve*)(operatingSystem=*erve*)))" name operatingSystem

# 3. Filter for computer objects containing the substring '2012' within the operatingSystem attribute, using paged searches (pr=1000)
ldapsearch -E pr=1000/noprompt -LLL -H ldap://10.1.4.111 -x -D "DOMAIN\USER" -w "MyPassword1" -b "DC=X,DC=Y,DC=AD" "(operatingSystem=*2012*)"

# 4. Identify user objects within the specified directory that contain an active os400Password attribute
ldapsearch -E pr=1000/noprompt -LLL -H ldap://10.1.4.11 -x -D "DOMAIN\USER" -w "MyPassword1" -b "DC=X,DC=Y" "(&(objectCategory=person)(objectClass=user)(os400Password=*))"
```

_Note on Query 4:_ The presence of the `os400Password` attribute indicates specialized legacy integrations, such as IBM AS/400 systems or other legacy backends that manage credentials through this specific field.

```
# 5. Retrieve user accounts configured with Kerberos Service Principal Names (SPN) matching a pattern, while excluding krbtgt and disabled accounts
ldapsearch "(&(samAccountType=805306368)(servicePrincipalName=M*SS*L/*)(!samAccountName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))" samAccountName,pwdLastSet
```

6. **OPSEC Optimization:** Rewrite the previous query filter using hex encoding format to cleanly bypass string-based detection filters monitoring for common SPN keywords.


```
# 7. Query for active service accounts with SPNs matching alphabetic ranges from 'a' to 'z', excluding disabled accounts and krbtgt
ldapsearch "(&(samAccountType=805306368)(servicePrincipalName>=a)(servicePrincipalName<=z)(!samAccountName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))" samAccountName,pwdLastSet

# Extract comprehensive profile metadata for a specific user using their userPrincipalName
ldapsearch -x -H ldap://yourdomain.com -D "cn=admin,dc=yourdomain,dc=com" -w "password" -b "dc=yourdomain,dc=com" "(&(objectClass=user)(userPrincipalName=username))" lastLogonTimestamp pwdLastSet whenChanged whenCreated nTSecurityDescriptor lastLogon lastLogOff

# Target an individual account profile to extract control flags, group memberships, and log timelines with a paged limit of 1,000 entries
ldapsearch -E pr=1000/noprompt -LLL -H ldap://10.10.10.10 -x -D "ZZZZZZ\\yyyyyy" -W -b "DC=x,DC=y" "(userPrincipalName=yyyyyyy@zzzzzz.com)" sAMAccountName userAccountControl whenChanged whenCreated pwdLastSet objectCategory memberOf lastLogon lastLogoff

# Check for Kerberoastable Users: Target active accounts with SPNs between 'a' and 'z', excluding disabled accounts and specific hex string matches
ldapsearch -E pr=1000/noprompt -LLL -H ldap://10.10.10.10 -x -D "zzz\\yyy" -W -b "DC=zzz,DC=local" "(&(samAccountType=805306368)(servicePrincipalName >= a)(servicePrincipalName <= z)(!samAccountName=HEX STRING)(!(UserAccountControl:1.2.840.113556.1.4.803:=\32)))" sAMAccountName pwdLastSet

# Filter for active computer objects running server operating systems while explicitly excluding disabled machine accounts
ldapsearch -H ldap://10.10.10.10 -x -D "zzz@yyy.com" -w "xxxx" -b "dc=yyy,dc=local" "(&(objectCategory=computer)(operatingSystem=*server*)(!userAccountControl:1.2.840.113556.1.4.803:=8192))"

# Search for user objects with an assigned SPN starting between 'a' and 'z', excluding krbtgt and disabled users, returning account names and password age
ldapsearch -H ldap://10.10.10.10 -x -D "zzz@yyy.com" -w "xxxx" -b "dc=yyy,dc=local" "(&(samAccountType=805306368)(servicePrincipalName >=a)(servicePrincipalName <= z)(!(samAccountName=HEX STRING)(!(UserAccountControl:1.2.840.113556.1.4.803:=\32))))" sAMAccountName pwdLastSet
```

Convert 18-digit Active Directory/LDAP timestamps to human-readable dates using an epoch converter to accurately map out account aging.

#### LDAP Queries Reference List:

```
ldapsearch -x -H ldap://yourdomain.com -D "cn=admin,dc=yourdomain,dc=com" -w "password" -b "dc=yourdomain,dc=com" "(&(objectClass=user)(userPrincipalName=username))" lastLogonTimestamp pwdLastSet whenChanged whenCreated nTSecurityDescriptor lastLogon lastLogOff
```

### Credentials Hunting on Enumerated Shares

- Cross-reference the server inventory compiled during your LDAP enumeration to verify your access rights and locate high-value files.
- If fresh credentials are uncovered, immediately refresh your security token context and re-run your network share discovery process to locate newly accessible assets.
- **Manual Share Browsing:** Remember that network paths ending with a `$` character indicate hidden administrative shares that are completely omitted from standard graphical user interface views.
- **Automated Share Enumeration:** Deploy our proprietary internal red team tool to systematically map and recursively enumerate accessible shares at randomized intervals to minimize network traffic anomalies.

### Credentials Verification

When fresh credentials are recovered, audit the account's `lastLogon` and `pwdLastSet` attributes to accurately evaluate whether the credentials are valid and active. If the credentials appear valid, keep in mind that the organization may enforce Multi-Factor Authentication (MFA/2FA) on Microsoft 365 portals. To validate the credentials safely from an internal perspective:

1. Validate the credentials by executing a highly targeted, single-object custom LDAP query, which is significantly more stealthy.
2. **OR (Not Recommended):** Test the credentials against a network share that your current user context already has permission to view. This approach is not recommended because SMB authentication generates highly visible and heavily monitored authentication telemetry. To safely list available shares on a target server: `smbclient -L //servertarget -U corp/usermarchus`.

If the credentials are functional, re-enumerate the network shares using execution utilities like `runas` or via an RDP session to identify newly exposed files and directories.

### Cloud and Content Repository Auditing

- Audit accessible corporate cloud storage environments including OneDrive and SharePoint.
- Systematically check for publicly accessible documents shared globally with the "Everyone" group across the enterprise SharePoint deployment.
- Navigate to the Office application search portal (`www.office.com/appSearch`) and filter systematically for high-value file extensions: `filetype:pdf`, `xlsx`, `ps1`, `doc`, `zip`, `txt`, `.xls`, `cmd`, `docx`, `bat`, etc.
- **Search Context:** Query target usernames or critical keywords like 'password', 'credential', or 'config'.

## What to do next? (You still don't have credentials)

If you already possess valid credentials, skip this section and proceed directly to Lateral Movement. If you do not have usable credentials, leverage the following techniques to harvest them or elevate privileges within the current host context:

- **LSASS Process Dumping:** This is a highly monitored, high-telemetry operation. If SentinelOne (S1) is active on the host endpoint, standard LSASS memory dumping techniques will be blocked instantly.
- **PPL Protection Evasion:** If SentinelOne is active and the Windows host lacks recent updates, you can leverage PPLFault (`https://github.com/gabriellandau/PPLFault`) to exploit page faults, bypass PPL protections, and dump process memory without triggering EDR detections.
- **Premium Tooling:** Execute specialized Beacon Object Files (BoFs) from premium commercial offensive sets, such as the Outflank Security Tooling suite.
- **LAPS Auditing:** Verify if LAPS is deployed across the environment to evaluate local administrator architecture.
- **ADCS Exploitation:** Target Active Directory Certificate Services (ADCS) misconfigurations to obtain a Domain Admin (DA) context via vulnerable certificate templates.
    1. Run RegCertipy or execute equivalent Outflank BoF tools.
    2. Analyze the ADCS enumeration output to identify vulnerable misconfigurations (e.g., ESC1 through ESC8).
- **Kerberos Attacks:** Enumerate the SPN list and perform Kerberoasting. We utilize a custom, proprietary BoF rewritten by our internal red team to minimize telemetry. Crack the recovered tickets offline using Hashcat: `hashcat -m 13100 -a 0 /home/1/output.txt`.
- **AS-REP Roasting:** Perform AS-REP Roasting against accounts that do not require Kerberos preauthentication.
- **Delegation Auditing:** Analyze constrained, unconstrained, or resource-based constrained delegations on objects you control.
- **Stale Account Targeting:** Audit stale user accounts, profiles with local administrative privileges, objects with descriptive fields populated, or accounts where the password was last set years ago.
- **SOAPHound Auditing:** Run SOAPHound (`https://github.com/FalconForceTeam/SOAPHound`). Use the data in the BloodHound Community edition to execute custom Cypher queries, such as: `MATCH (n:User) WHERE n.lastlogontimestamp < (datetime().epochseconds - (720 * 86400)) AND n.enabled=TRUE RETURN n`. This identifies enabled user objects that have not authenticated in over 720 days.
- **In-Memory BloodHound Collection:** Execute BloodHound collection stealthily via specialized in-memory BoFs instead of running raw SharpHound binaries on disk.
- **NTLM Relaying:** Perform NTLM Relaying; refer to standard farming methodologies (`https://www.mdsec.co.uk/2021/02/farming-for-red-teams-harvesting-netntlm/`). Deploy an internal NTLM relay architecture using tools like Farmer (`https://github.com/mdsecactivebreach/Farmer`), executed via the `InlineExecute-Assembly` BoF (`https://github.com/anthemtotheego/InlineExecute-Assembly`). This method loads the required DLL into memory and passes the .NET assembly directly, executing it cleanly without writing binaries to disk.
- **Social Engineering & Helpdesk Phishing:** Alternatively, use a "shotgun approach" as an operational last resort: simulate a technical fault on your provided platform and contact the organization's IT helpdesk. There is a high likelihood that helpdesk personnel will establish an RDP session or log directly into your desktop to troubleshoot, creating a premier opportunity to dump LSASS or harvest administrative session credentials once they authenticate. If permitted in the rules of engagement, vishing is a highly effective tactic since you possess a legitimate corporate identity. Craft a solid pretext to escalate privileges; review the Office 365 organizational chart to identify if a RACI matrix exists, determining the precise personnel responsible for approving security group modifications. Execute social engineering by contacting support with a well-crafted pretext, such as: _"I am a new employee and my account permissions were misconfigured. Manager X is aware and instructed me to contact support to mirror the exact access groups as Employee Y in my department."_ The new-employee pretext is highly reliable because the customer-provided account was recently instantiated, fitting perfectly into the operational narrative.
    
**Useful Tools for the Operation:** We maintain custom BoFs developed by our internal red team, including tools that interact with vulnerable kernel drivers that are not publicly disclosed. Utilize HiddenDesktop (`https://github.com/WKL-Sec/HiddenDesktop`), which enables an operator to establish a concurrent interactive desktop session via RDP even if another active user is currently logged in (Note: standard Windows client OS installations restrict concurrent sessions to a single user, whereas Windows Server environments manage multiple sessions via Remote Desktop Services licensing).

## Lateral Movement (You Have Credentials)

Once credentials are confirmed and you have mapped out newly accessible SMB shares, proceed with the following lateral movement techniques:

1. **Custom RPC Remote Lateral Movement:** Execute lateral movement via custom RPC endpoints, tailoring your service parameters based on the data gathered during the initial host recon phase to maintain high OPSEC.
2. **Manual Service Creation:** Execute lateral movement by completely constructing and registering a new service manually, then triggering its execution to avoid standard tool signatures.
3. **Custom Service Execution Binary:** Leverage modified implementations like CSExec (`https://github.com/malcomvetter/CSExec`), which replicates PsExec functionality but requires customization to evade modern behavioral heuristics.

### Beacon Staging Strategy

This operational phase typically occurs around day 8 of the engagement. Given a standard 90-day assessment timeline, leverage the initial phase to move deliberately and slowly to avoid triggering proactive SOC detections.

## Machine Setup

1. Start the command-and-control team server.
2. Create an HTTP beacon payload configuration.
3. Export the raw beacon payload configuration as a binary file.
4. Select the appropriate configuration from the Arsenal Kit to apply a sleep mask and prepend a User-Defined Reflective Loader (UDRL). When the Cobalt Strike beacon calls its sleep routine, the hook intercepts the execution flow and can perform three operations:
    
    - Call standard Windows classic sleep (highly detected by EDR via thread stack analysis).
    - Call `WaitForSingleObject`, blocking the process thread for a specific timeout interval.
    - Utilize advanced sleep techniques like Ekko or 5Spider; while in a sleeping state, these techniques configure application timers called every 100 ms to execute an `NtContinue` function. This essentially performs no operations but maintains thread continuity, allowing you to pass and obfuscate the system registry context.
5. Run `build.bat \path\to\ 1.bin` to compile.
    

### Encrypt the Beacon

Apply two independent layers of encryption. The first layer encrypts the core beacon payload to obscure embedded Indicators of Compromise (IoCs). The second encryption stage wraps the payload to mask the UDRL configuration entirely. The resulting artifact behaves like a raw, benign data blob that performs no immediate malicious API calls, registering only as legitimate operations during static inspection.

Upon execution, choose your ingestion path based on your operational scenario. From the CLI, the payload can be configured to open a named pipe or accept a specific parameter to read directly from a file. This gives you two distinct options: reading directly from an on-disk asset or awaiting data arrival via an established named pipe.

Because the named pipe string is uniquely randomized across the two encryption layers, every generated beacon payload yields a completely unique cryptographic hash. This renders static file signatures entirely useless for defenders. The loader transfers files via simple in-memory stream copies.

Be mindful that launching commands via standard process creation is logged by modern telemetry, exposing the passed parameters to threat hunters. Your localized workspace payload structure should consist of:

- `1.bin`
- `pipe.txt`
- `a.dat`
- `teamsupdater.exe`

Generate a uniquely structured payload for every target system to prevent an EDR from flagging a uniform signature across the environment, which would compromise your entire persistence footprint simultaneously.

## How It Works

Cobalt Strike Beacons leveraging the SMB protocol operate as bind shells, whereas standard HTTP beacons establish reverse shell connections back to the redirector infrastructure.

### Cobalt Strike Beacon Execution Sequence

1. Generate the payload as a raw binary artifact (without a standard file extension).
2. Rename the target executable to a `.dll` format.
3. Transfer the binary to an OPSEC-compliant folder and assign it a legitimate name that mirrors native OS components.

A raw Cobalt Strike beacon contains: `UDRL + PADDING (e.g., spaces) + Encryption Key + Beacon Payload`. When reviewed in a hex editor, standard x64 reflective payloads natively begin with the byte sequence `48 89 5C`. To neutralize signature-based detections targeting this specific header, we apply an additional layer of custom encryption and memory patching.

The internal beacon payload is encrypted using RC4. When Cobalt Strike compiles the beacon, it embeds a distinct decryption key; inspecting the artifact with a hex editor reveals both the overall payload size and this specific key string.

Our internal red team launcher ingests the `.dll`, maps it directly into virtual memory space, and resolves execution via a named pipe or localized file reference. While typical commodity malware pulls components directly across the internet, our localized configuration remains significantly more covert.

Named pipes provide an inter-process communication (IPC) mechanism allowing process A and process B to securely exchange data across a shared virtual memory boundary. Malicious loaders often leverage named pipes alongside specific named mutexes to enforce single-instance execution, ensuring the payload does not run concurrently on the same host. Mutexes function as core synchronization objects for active execution threads.

### How a Beacon Functions

The beacon remains in a low-telemetry sleep state until its internal timer expires. It wakes up to check the command-and-control server for queued tasks issued by the operator, executes any received instructions, and immediately returns to its sleep state.

We enforce a sleep configuration with a specific "jitter" percentage. Jitter adds a randomized offset to the callback interval, disrupting the rigid timing patterns that network anomaly detection engines look for. This prevents defenders from establishing network IoCs based on fixed beacon frequencies. A baseline sleep interval of 27 seconds with jitter is highly effective for active operations. During non-operational nighttime hours, always increase the sleep interval significantly to minimize your exposure footprint.

## How ETW and EDRs like SentinelOne Function, and Evasion Basics

Event Tracing for Windows (ETW) functions essentially as a high-performance event tracking system comprising two main entities:

1. **Producer Processes:** These actively generate event data.
2. **Consumer Processes:** These read and ingest these event streams.

Standard ETW components run in userland and rely on interactions with heavily monitored Windows APIs. ETW Patching operates similarly to an AMSI bypass. By modifying the execution flow of specific userland logging functions to return instantly, you prevent the caller process from writing telemetry to the ETW subsystem. As a result, the consumer processes receive no event data.

However, during initial process creation, event logging is controlled kernel-side rather than in userland, utilizing kernel callbacks to track process execution. Evading these indicators requires operating at the kernel level (such as using tools from the Outflank set), which necessitates local administrator privileges on the compromised host.

Modern EDR platforms, particularly SentinelOne (S1), monitor process instantiation driver-side via kernel callbacks. Simultaneously, they perform userland DLL injection into new processes to intercept execution and track state modifications, effectively acting as a monitoring proxy over original system libraries.

Every process initialized in memory maps a structural list of its loaded modules. Inspecting these structures reveals the execution libraries called during initialization; the primary executable loads first, followed by baseline system libraries like `ntdll.dll`, `kernel32.dll`, or `kernelbase.dll`. In environments protected by SentinelOne, S1's monitoring DLL is injected directly alongside these libraries.

While some malware attempts to parse these module structures to calculate API offsets and jump directly to system calls, they are often detected because they inadvertently land within memory regions hooked or managed by S1, triggering behavioral alerts. When executing an unhooked API call via custom C code or direct memory indexing, jumping blindly can inadvertently route execution back through an S1-monitored boundary, leading to detection. To bypass this entirely, operators must leverage kernel-side techniques or direct system calls to ensure thread creation and execution generate no usable userland telemetry for defenders.