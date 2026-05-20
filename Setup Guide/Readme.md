## Phase 1  Endpoint Instrumentation
1.1 Install Sysmon on Windows Client
Sysmon provides detailed process, network, and file telemetry beyond what Windows Event Logs natively capture. Download Sysmon from Microsoft Sysinternals and deploy with a configuration file (SwiftOnSecurity config recommended).
powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml

Verify Sysmon is running:
powershell 
Get-Service Sysmon64

Sysmon logs to Microsoft-Windows-Sysmon/Operational in Event Viewer. Event ID 1 (Process Create) is the primary event used in this lab for detecting Mimikatz execution.

1.2 Install Wazuh Agent on Windows Client
Deploy the Wazuh agent and point it to the Wazuh Manager IP.
After installation, edit C:\Program Files (x86)\ossec-agent\ossec.conf to ingest Sysmon logs. Add the following localfile block:


<ossec_config>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</ossec_config>

Restart the Wazuh agent service:
powershell 
Restart-Service -Name WazuhSvc

## Phase 2  Wazuh Manager Configuration
2.1 Enable Log Archiving
By default Wazuh only stores alerts, not all raw logs. To enable full log archiving, edit /var/ossec/etc/ossec.conf on the Wazuh Manager:

<ossec_config>
  <global>
    <logall>yes</logall>
    <logall_json>yes</logall_json>
  </global>
</ossec_config>

2.2 Enable Filebeat Archiving
Edit /etc/filebeat/filebeat.yml on the Wazuh Manager and set archives to true:

filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: true

Restart Filebeat:
sudo systemctl restart filebeat

2.3 Create Wazuh Archives Index
In the Wazuh dashboard:
- Go to Stack Management → Index Patterns
- Create a new index pattern: wazuh-archives-*
- Set the time field to @timestamp
- Use Discover and select the wazuh-archives-* index to search all ingested logs including non-alert events


## Phase 3 Mimikatz Detection
3.1 Simulate Mimikatz Execution
Download Mimikatz on the Windows client and execute it. The Wazuh agent forwards the Sysmon Event ID 1 (Process Create) log to the Wazuh Manager. Verify the event appears in the Wazuh dashboard under the wazuh-archives-* index.

3.2 Create Custom Detection Rule
The default Wazuh ruleset does not alert specifically on Mimikatz. A custom rule targeting the OriginalFileName field in Sysmon Event ID 1 is used, this catches Mimikatz even if an attacker renames the binary.

In the Wazuh dashboard go to Management → Rules → Add new rule, or edit /var/ossec/etc/rules/local_rules.xml directly:

<group name="sysmon,">

  <rule id="100002" level="15">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
    <description>Mimikatz usage detected - OriginalFileName match</description>
    <mitre>
      <id>T1003</id>
    </mitre>
  </rule>

</group>

Restart Wazuh Manager to load the new rule:
sudo systemctl restart wazuh-manager

3.3 Validate the Rule
we rename mimikatz.exe to youareawesome.exe and execute it. The rule still fired because it matches on OriginalFileName (embedded in the PE header), not the filename on disk. we confirmed the alert appears in the Wazuh dashboard with description "Mimikatz usage detected - OriginalFileName match".

## Phase 4  TheHive Setup
4.1 Cassandra Configuration
Edit /etc/cassandra/cassandra.yaml:
cluster_name: 'mydfir'

listen_address: 10.0.0.5
rpc_address: 10.0.0.5

seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "10.0.0.5"

Important: we use the private IP, not the public IP. Cassandra cannot bind to a public IP that is NAT'd at the network layer (common in Azure/AWS). Using the public IP results in:
Fatal configuration error: Unable to bind to address /20.x.x.x:7000

Start Cassandra and verify port 9042 is listening before proceeding:
bash sudo systemctl start cassandra
ss -tlnp | grep 9042

4.2 Elasticsearch Configuration
Edit /etc/elasticsearch/elasticsearch.yml:

cluster.name: thehive
node.name: node-1
network.host: 10.0.0.5
http.port: 9200
discovery.type: single-node

Limit JVM heap to prevent OOM kills. Create /etc/elasticsearch/jvm.options.d/heap.options:
-Xms1g
-Xmx1g

Start Elasticsearch and verify:
sudo systemctl start elasticsearch
ss -tlnp | grep 9200

4.3 TheHive Configuration
our public ip: 20.63.88.61

Edit /etc/thehive/application.conf:
hocondb.janusgraph {
  storage {
    backend = cql
    hostname = ["10.0.0.5"]
    cql {
      cluster-name = mydfir
      keyspace = thehive
    }
  }
  index.search {
    backend = elasticsearch
    hostname = ["10.0.0.5"]
    index-name = thehive
  }
}

storage {
  provider = localfs
  localfs.location = /opt/thp/thehive/files
}

application.baseUrl = "http://20.63.88.61:9000"
http.address = "0.0.0.0"
http.port = 9000

Start services in order and wait between each:
bash sudo systemctl start cassandra
sleep 60
sudo systemctl start elasticsearch
sleep 30
sudo systemctl start thehive
Access TheHive at http://20.63.88.61:9000. 
Default credentials: admin@thehive.local / secret. Change immediately after first login.

4.4 TheHive Organisation and User Setup

Log in as admin
Create organisation: mydfir
Create analyst user: mydfir@test.com, profile: Analyst
Create SOAR service account: shuffle@test.com, profile: Analyst
Generate an API key for the shuffle@test.com account, this is used to authenticate Shuffle to TheHive

## Phase 5  Shuffle SOAR Workflow
5.1 Connect Wazuh to Shuffle
Add an integration block to /var/ossec/etc/ossec.conf on the Wazuh Manager:

<ossec_config>
  <integration>
    <name>shuffle</name>
    <hook_url>https://shuffler.io/api/v1/hooks/https://shuffler.io/api/v1/hooks/webhook_2d94fa1e-a67b-46d9-b042-eefb31a10e21</hook_url>
    <rule_id>100002</rule_id>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>

Restart the Wazuh Manager:
bash sudo systemctl restart wazuh-manager
Only alerts matching rule 100002 (the Mimikatz rule) are forwarded to Shuffle.

5.2 Workflow Design
Webhook (Wazuh trigger)

    |
    
Regex: Extract SHA256 hash from alert
    
    |

VirusTotal: Get hash report
    
    |

    
TheHive: Create alert
    
    |

    
Email: Notify SOC analyst


5.3 SHA256 Extraction via Regex
In the Shuffle workflow, add a Regex Capture Group action on the raw Wazuh alert JSON.
The SHA256 hash is in win.eventdata.hashes.
We use this regex to extract it:
SHA256=([A-Fa-f0-9]{64})
Reference the captured group in subsequent workflow steps as the hash value passed to VirusTotal.
5.4 VirusTotal Integration
Add the VirusTotal app to the workflow. Configure:

Action: Get a hash report
Hash: output of the regex capture step
API Key: your VirusTotal account API key

The output includes last_analysis_stats with malicious, suspicious, and harmless vote counts from all AV engines.
5.5 TheHive Alert Creation

Method: POST
URL: http://20.63.88.61:9000/api/v1/alert
Headers:

json{
  "Content-Type": "application/json",
  "Authorization": "Bearer <our shuffle api key>"
}

Body:

json{
  "title": "Mimikatz Usage Detected",
  "description": "Mimikatz execution detected on endpoint. VirusTotal score: $virustotal.body.data.attributes.last_analysis_stats.malicious malicious detections.",
  "type": "Internal",
  "source": "Wazuh",
  "sourceRef": "$exec.id",
  "severity": 3,
  "tlp": 2,
  "pap": 2,
  "flag": false,
  "status": "New",
  "tags": ["T1003", "Mimikatz", "Credential Access"]
}

$exec.id generates a unique sourceRef per execution, required by TheHive.

5.6 Email Notification
Add the Email app to the workflow connected after VirusTotal. 
Configure:

To: SOC analyst email address
Subject: [ALERT] Mimikatz Detected: Immediate Investigation Required
Body:

Mimikatz execution has been detected on an endpoint.
Time: $exec.text.win.eventdata.utcTime
Title: $exec.title
Host: $exec.text.win.system.computer
user: $exec.text.win.eventdata.user
hash: $sha256_regex.group_0
virustotal detections: $virustotal.#.body.data.attributes.last_analysis_stats.malicious

A case has been created in TheHive for investigation.
Please log in and begin triage immediately.

## Phase 6 End-to-End Validation

Execute Mimikatz (or the renamed variant) on the Windows client
Confirm Sysmon Event ID 1 is generated on the endpoint
Confirm the alert triggers rule 100002 in the Wazuh dashboard
Confirm the alert appears in Shuffle as a new workflow execution
Confirm the SHA256 hash is correctly extracted by the regex
Confirm VirusTotal returns a hash report with detection stats
Confirm a new alert is created in TheHive under the mydfir organisation, visible to the mydfir@test.com analyst account
Confirm the SOC analyst receives the email notification

## Phase 7 Automated Response Actions
When the SOC analyst reviews the alert in TheHive and determines it is a true positive, they trigger a response action. TheHive notifies Shuffle via webhook, which calls the Wazuh Manager API to execute active response commands on the Windows endpoint. Three response actions are implemented: kill the malicious process, quarantine the file, and block the hash via Windows Firewall.

7.1 Wazuh Active Response Configuration
Wazuh active response works by defining a command (a script on the agent) and an active-response block that maps the command to a trigger condition. For custom response actions, scripts are placed on the agent at:

Windows: C:\Program Files (x86)\ossec-agent\active-response\bin\

Edit /var/ossec/etc/ossec.conf on the Wazuh Manager to define the response commands:
xml<ossec_config>

  <!-- Kill malicious process by name -->
  <command>
    <name>kill-process</name>
    <executable>kill-process.cmd</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <!-- Quarantine (move) malicious file -->
  <command>
    <name>quarantine-file</name>
    <executable>quarantine-file.cmd</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <!-- Block hash via Windows Firewall -->
  <command>
    <name>block-hash</name>
    <executable>block-hash.cmd</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <!-- Active response bindings -->
  <active-response>
    <command>kill-process</command>
    <location>local</location>
    <rules_id>100002</rules_id>
  </active-response>

  <active-response>
    <command>quarantine-file</command>
    <location>local</location>
    <rules_id>100002</rules_id>
  </active-response>

  <active-response>
    <command>block-hash</command>
    <location>local</location>
    <rules_id>100002</rules_id>
  </active-response>

</ossec_config>
Restart Wazuh Manager after editing:
bash sudo systemctl restart wazuh-manager

7.2 Agent-Side Response Scripts
The following scripts are placed on the Windows endpoint at C:\Program Files (x86)\ossec-agent\active-response\bin\.
Wazuh passes alert data to the script as a JSON object via stdin. The scripts parse the relevant fields and execute the remediation.

kill-process.cmd: Terminates the malicious process by image name extracted from the alert

batch@echo off
setlocal enabledelayedexpansion

:: Read alert JSON from stdin passed by Wazuh
set /p ALERT_JSON=

:: Extract process name from alert data (win.eventdata.image)
:: Use PowerShell to parse the JSON and kill the process

powershell -Command ^
  "$alert = '%ALERT_JSON%' | ConvertFrom-Json; ^
   $procPath = $alert.parameters.alert.data.win.eventdata.image; ^
   $procName = Split-Path $procPath -Leaf; ^
   Get-Process | Where-Object { $_.MainModule.FileName -like '*' + $procName } | Stop-Process -Force; ^
   Write-Output ('Killed process: ' + $procName)" >> "%WINDIR%\Temp\ar-kill-process.log" 2>&1

quarantine-file.cmd: Moves the malicious file to an isolated quarantine directory

batch@echo off
setlocal enabledelayedexpansion

set QUARANTINE_DIR=C:\Quarantine
if not exist "%QUARANTINE_DIR%" mkdir "%QUARANTINE_DIR%"

set /p ALERT_JSON=

powershell -Command ^
  "$alert = '%ALERT_JSON%' | ConvertFrom-Json; ^
   $filePath = $alert.parameters.alert.data.win.eventdata.image; ^
   if (Test-Path $filePath) { ^
     $dest = 'C:\Quarantine\' + (Split-Path $filePath -Leaf) + '_quarantined_' + (Get-Date -Format yyyyMMdd_HHmmss); ^
     Move-Item -Path $filePath -Destination $dest -Force; ^
     Write-Output ('Quarantined: ' + $filePath + ' -> ' + $dest) ^
   } else { ^
     Write-Output ('File not found: ' + $filePath) ^
   }" >> "%WINDIR%\Temp\ar-quarantine.log" 2>&1

block-hash.cmd: Blocks execution of the file by SHA256 hash using Windows Defender and adds a firewall rule to block outbound connections from the process path
batch@echo off
setlocal enabledelayedexpansion

set /p ALERT_JSON=

powershell -Command ^
  "$alert = '%ALERT_JSON%' | ConvertFrom-Json; ^
   $hashes = $alert.parameters.alert.data.win.eventdata.hashes; ^
   $sha256 = ($hashes -split ',') | Where-Object { $_ -like 'SHA256=*' }; ^
   $sha256 = $sha256 -replace 'SHA256=', ''; ^
   $filePath = $alert.parameters.alert.data.win.eventdata.image; ^
   Add-MpPreference -ExclusionPath $filePath; ^
   Add-MpThreatCatalog; ^
   netsh advfirewall firewall add rule name='BLOCK_MALICIOUS_$sha256' dir=out action=block program='$filePath' enable=yes; ^
   Write-Output ('Blocked hash: ' + $sha256 + ' | File: ' + $filePath)" >> "%WINDIR%\Temp\ar-block-hash.log" 2>&1

All three scripts log their output to C:\Windows\Temp\ for forensic review and audit trail.

7.3 Trigger Response from Shuffle via Wazuh API
Rather than relying solely on automatic rule-based active response, the SOC analyst can trigger the response manually from TheHive. TheHive fires a webhook to a second Shuffle workflow on analyst action. A email with a link to execute the automated response is sent to the analyst.
In Shuffle, we create a Response Workflow with a webhook trigger. When the analyst marks the TheHive alert as confirmed and clicks respond, TheHive calls this webhook. Shuffle then calls the Wazuh Manager API to execute the active response on the specific agent.

Shuffle HTTP action: trigger active response via Wazuh API:

Method: POST
URL: https://we put our wazh manager ip here/active-response
Headers:

json{
  "Content-Type": "application/json",
  "Authorization": "Bearer <We put our wazuh JWT api token>"
}

Body:

json{
  "command": "kill-process0",
  "custom": false,
  "alert": {
    "data": {
      "win": {
        "eventdata": {
          "image": "$thehive_alert.data.win.eventdata.image",
          "hashes": "$thehive_alert.data.win.eventdata.hashes"
        }
      }
    }
  },
  "arguments": [],
  "wait_for_complete": true,
  "agents_list": ["$thehive_alert.agent_id"]
}
Obtain the Wazuh API JWT token before making the active response call:
bash curl -u <wazuh_api_user>:<password> -k -X POST \
  "https://20.200.125.36::55000/security/user/authenticate" \
  | python3 -m json.tool
Use the returned token value as the Bearer token in the Authorization header.

Repeat the Shuffle HTTP action for each response command (quarantine-file0, block-hash0), connecting them all to the wazuh_auth so all three execute on confirmation.

7.4 Verify Response Execution
On the Windows client, confirm each action completed:
powershell# Confirm process was killed
Get-Process | Where-Object { $_.Name -like "*mimikatz*" }
# Should return nothing

Get-ChildItem "C:\Windows\Temp\" | Where-Object { $_.Name -like "ar-*" }


# Confirm file was quarantined
Test-Path "C:\Quarantine\"
Get-ChildItem "C:\Quarantine\"

# Confirm firewall rule was created
netsh advfirewall firewall show rule name=all | findstr "BLOCK_MALICIOUS"

# Review active response logs
Get-Content "C:\Windows\Temp\ar-kill-process.log"
Get-Content "C:\Windows\Temp\ar-quarantine.log"
Get-Content "C:\Windows\Temp\ar-block-hash.log"

On the Wazuh Manager, active response executions are logged at:
bash sudo tail -f /var/ossec/logs/active-responses.log

##  Phase 8 — End-to-End Validation (Full Pipeline)

Execute Mimikatz on the Windows client
Confirm Sysmon Event ID 1 captured and forwarded to Wazuh Manager
Confirm rule 100002 fires in Wazuh dashboard
Confirm Shuffle receives the alert via webhook and executes the detection workflow
Confirm SHA256 hash extracted correctly by regex
Confirm VirusTotal returns malicious detection count
Confirm TheHive alert created under mydfir organisation
Confirm SOC analyst email received with hash, VT score and link to execute automated response
Confirm Shuffle response workflow executes all three active response commands
Confirm on Windows client: process killed, file in quarantine, firewall rule present
Confirm Wazuh active response log records all three executions
Confirm analyst receives response confirmation email


Troubleshooting Reference

1. Cassandra fails to start with bind error
Root cause: listen_address set to public IP
fix: Set to private IP in cassandra.yaml

2. Shuffle cannot reach TheHive (port 9000 refused)
root cause: Azure NSG blocking inbound traffic
fix: Add inbound rule for TCP port 9000 in NSG

3. Archives index not showing all logs
   root cause: Filebeat archives disabled
   fix: Set archives.enabled: true in filebeat.yml



MITRE ATT&CK Coverage

Technique: OS Credential Dumping  
ID:T1003
Detection Method: Sysmon Event ID 1 + OriginalFileName match
Response Action: Kill process, quarantine file, block hash


Technique: Credential Dumping: LSASS Memory
ID: T1003.001
Detection Method: Sysmon process create with Mimikatz signature
Response Action: Kill process, quarantine file


Technique: Indicator Removal
ID: T1070
Detection Method: Hash-based detection survives binary rename
Response Action: Block hash via firewall rule

Security Hardening Notes

Rotate the default TheHive admin@thehive.local password immediately post-deployment
Scope the Azure NSG inbound rule for port 9000 to known IPs only, do not leave it open to 0.0.0.0/0 in production
Store API keys (VirusTotal, TheHive SOAR account) as Shuffle secrets, not plaintext in workflow steps
The TheHive SOAR service account (shuffle@test.com) should have minimum required permissions, Analyst profile is sufficient for alert creation
Enable HTTPS on TheHive using an nginx reverse proxy before any production use
Do not commit application.conf, ossec.conf, or any file containing API keys or IP addresses to a public repository

