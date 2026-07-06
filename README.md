# ELK Security Monitoring Setup 

This repository contains the setup documentation and configurations for deploying a localized ELK (Elasticsearch, Logstash, Kibana) Stack environment for security monitoring. 

**Author**: Usman Ibrahim

## Project Objectives
1. Deploy ELK Stack locally on Windows.
2. Configure Fleet Server and enroll a Linux Elastic Agent.
3. Verify logs from Fleet Server and Linux Agent in Kibana.
4. Integrate Windows Defender with ELK.
5. Integrate Sysmon and monitor security events.
6. Integrate YARA with ELK and develop custom YARA rules.
7. Validate alert generation and visibility in Kibana.

---

## Architecture Overview
- **Host VM**: Windows 10 (Runs Elasticsearch, Kibana, Fleet Server, Windows Defender, Sysmon, YARA)
- **Endpoint VM**: Ubuntu Linux (Runs Elastic Agent)

---

## Phase 1: Deploy ELK Stack locally on Windows

1. **Download the Elastic Stack:**
   - Downloaded [Elasticsearch v8.14.3](https://www.elastic.co/downloads/elasticsearch) (Windows zip) and [Kibana v8.14.3](https://www.elastic.co/downloads/kibana) (Windows zip).
   - Extracted to `C:\ELK\elasticsearch` and `C:\ELK\kibana`.

2. **Start Elasticsearch:**
   - Opened Command Prompt as Administrator, navigated to `C:\ELK\elasticsearch\bin`, and ran `elasticsearch.bat`.
   - Saved the auto-generated elastic user password and the Kibana enrollment token.

3. **Start Kibana:**
   - Opened a second Command Prompt, navigated to `C:\ELK\kibana\bin`, and ran `kibana.bat`.
   - Accessed `http://localhost:5601`, pasted the enrollment token, and logged in using the `elastic` user credentials.

![Elasticsearch Setup](screenshots/Screenshot%201%20.png)
*(Elasticsearch and Kibana initialized via Command Prompt)*

![Kibana Welcome Page](screenshots/Screenshot%202.png)
*(Successful Kibana web interface initialization)*

---

## Phase 2: Configure Fleet Server and enroll a Linux Elastic Agent

### 1. Set up Fleet Server (Windows 10)
1. In Kibana, navigated to **Management > Fleet**.
2. Created a new Fleet Server Policy and generated a Service Token.
3. Installed the Elastic Agent on Windows 10 via PowerShell:
   ```powershell
   $ProgressPreference = 'SilentlyContinue'
   Invoke-WebRequest -Uri https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.14.3-windows-x86_64.zip -OutFile elastic-agent-8.14.3-windows-x86_64.zip
   Expand-Archive .\elastic-agent-8.14.3-windows-x86_64.zip
   cd elastic-agent-8.14.3-windows-x86_64
   .\elastic-agent.exe install --fleet-server-es=https://192.168.X.X:9200 --fleet-server-policy=fleet-server-policy --fleet-server-port=8220 --insecure
   ```
*(Note: `--insecure` was used to bypass self-signed certificate restrictions in this local lab).*

### 2. Enroll Linux Endpoint (Ubuntu)
1. In Kibana Fleet, selected **Add Agent** to generate an enrollment command for a Linux Tar deployment.
2. Executed on the Ubuntu VM:
   ```bash
   curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.14.3-linux-x86_64.tar.gz 
   tar xzvf elastic-agent-8.14.3-linux-x86_64.tar.gz
   cd elastic-agent-8.14.3-linux-x86_64
   sudo ./elastic-agent install --url=https://192.168.X.X:8220 --enrollment-token=<TOKEN> --insecure
   ```

![Ubuntu Agent Enrollment](screenshots/Screenshot%204.png)
*(Linux Elastic Agent deployment on Ubuntu terminal)*

![Fleet Agents Healthy](screenshots/Screenshot%205.png)
*(Fleet Server Dashboard showing both Windows and Linux agents as Healthy)*

---

## Phase 3: Verify Logs in Kibana

To confirm data ingestion:
1. Navigated to **Analytics > Discover**.
2. Filtered logs by `agent.name`. 
3. Successfully visualized baseline telemetry from both the `DESKTOP-XXXX` (Windows) and `ubuntu` endpoints.

![Logs Verified](screenshots/Screenshot%206.png)
*(Discover tab showing live log ingestion from the Ubuntu Linux agent)*

---

## Phase 4: Integrate Windows Defender with ELK

1. **Configure Integration**: 
   - In Kibana, added the **Custom Windows Event Logs** integration.
   - Set the Event log channel name to: `Microsoft-Windows-Windows Defender/Operational`.
   - Assigned to the Windows Fleet Server Policy.
2. **Trigger Alert**:
   - Downloaded the EICAR test string via PowerShell to trigger a Defender block:
     ```powershell
     Invoke-WebRequest -Uri "https://secure.eicar.org/eicar.com.txt" -OutFile ".\eicar.txt"
     ```
3. **Verify**:
   - Searched Kibana Discover for `windows.defender` to visualize the successful malware detection log.

![Defender Logs](screenshots/Screenshot%207.png)
*(Kibana Discover tab highlighting a Windows Defender malware detection alert)*

---

## Phase 5: Integrate Sysmon & Monitor Security Events

1. **Deploy Sysmon**:
   - Downloaded Sysmon and the SwiftOnSecurity configuration baseline on Windows 10:
     ```powershell
     Invoke-WebRequest -Uri "https://live.sysinternals.com/Sysmon64.exe" -OutFile ".\Sysmon64.exe"
     Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile ".\sysmonconfig.xml"
     .\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
     ```
2. **Configure Integration**:
   - Added another **Custom Windows Event Logs** integration in Kibana.
   - Set Event log channel name to: `Microsoft-Windows-Sysmon/Operational`.
3. **Verify**:
   - Validated the continuous stream of `Process Create` and `Network connection detected` events in Kibana Discover.

![Sysmon Logs](screenshots/Screenshot%2010.png)
*(Kibana Discover tab displaying highly detailed Sysmon Process Creation events)*

---

## Phase 6: Integrate YARA with ELK & Develop Custom Rules

1. **Deploy YARA & Create Rule**:
   - Downloaded YARA for Windows.
   - Created a custom detection rule (`custom_rule.yar`):
     ```yara
     rule Catch_Suspicious_Text {
         meta:
             description = "Detects our custom suspicious string"
         strings:
             $sus_string = "SUPER_SECRET_MALWARE_STRING"
         condition:
             $sus_string
     }
     ```
2. **Generate Logs**:
   - Scanned a test file and routed the output to a log file:
     ```powershell
     C:\yara\yara64.exe C:\yara\custom_rule.yar C:\yara\fake_malware.txt >> C:\yara\yara_alerts.log
     ```
3. **Ingest to ELK**:
   - Due to PowerShell's default UTF-16LE encoding, the log was converted to UTF-8:
     ```powershell
     Get-Content C:\yara\yara_alerts.log | Out-File -FilePath C:\yara\yara_alerts_utf8.log -Encoding utf8
     ```
   - The file was then ingested into Kibana utilizing the manual **Upload a file** feature (Data Visualizer).
4. **Verify**:
   - Searched Kibana Discover for `Catch_Suspicious_Text` to visualize the successful YARA match.

![YARA Logs](screenshots/Screenshot%2012.png)
*(Kibana Discover capturing the custom YARA "Catch_Suspicious_Text" alert)*

---

## Phase 7: Validate Alert Generation and Visibility

Alert generation was successfully validated across three distinct vectors:
- **Known Malware**: Windows Defender caught and logged the EICAR test file.
- **System Telemetry**: Sysmon logged deep endpoint visibility (processes, network).
- **Pattern Matching**: YARA successfully matched custom malware signatures and reported them.
All alerts were centralized and visualized successfully in the Kibana interface.

---

## Troubleshooting Log

- **Missing `curl` on Ubuntu**: Addressed by running `sudo apt update && sudo apt install curl`.
- **Connection Timed Out (Port 8220)**: Addressed by opening Windows Defender Firewall:
  `New-NetFirewallRule -DisplayName "Fleet Server Port 8220" -Direction Inbound -LocalPort 8220 -Protocol TCP -Action Allow`
- **Invalid Enrollment Token**: Addressed by generating a fresh Linux Tar installation command in Kibana after rebuilding the Fleet Server.
- **YARA Log Encoding Failures**: Addressed by piping Windows PowerShell outputs to `-Encoding utf8` to bypass Kibana's rejection of UTF-16LE formatted logs.
