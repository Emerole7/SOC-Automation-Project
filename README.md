# SOC-Automation-Project
A home lab project demonstrating end-to-end SOC automation using open-source tools. This project integrates Wazuh (SIEM/EDR), Shuffle (SOAR), and TheHive (case management) to automate alert triage, enrichment, and response.

Overview
This project documents the build-out of a SOC automation pipeline using three open-source platforms:

Wazuh — SIEM and EDR
TheHive — Case management
Shuffle — SOAR (Security Orchestration, Automation and Response)

The lab simulates a real-world detection and response workflow: a Mimikatz execution on a Windows endpoint triggers a Wazuh alert, which is forwarded to Shuffle, enriched via VirusTotal, escalated to TheHive as a case, and notified to a SOC analyst by email, all automated.
 
Architecture

Windows Client (Sysmon + Wazuh Agent)
        |
        | (OSSEC events)
        v
Wazuh Manager (Ubuntu)
        |
        | (Integration webhook)
        v
Shuffle SOAR
        |
        |--- Extract SHA256 hash
        |--- Query VirusTotal (reputation score)
        |--- Create Alert in TheHive
        |--- Send email to SOC Analyst
        |
        | (SOC Analyst reviews alert in TheHive, triggers response)
        |
        v
Shuffle SOAR (Response Workflow)
        |
        |--- Kill malicious process on endpoint
        |--- Quarantine malicious file on endpoint
        |--- Block malicious hash via Windows Firewall
        v
Wazuh Manager (Active Response)
        |
        v
Windows Client (remediation executed)

