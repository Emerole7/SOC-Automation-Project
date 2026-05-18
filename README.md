# SOC-Automation-Project
A home lab project demonstrating end-to-end SOC automation using open-source tools. This project integrates Wazuh (SIEM/EDR), Shuffle (SOAR), and TheHive (case management) to automate alert triage, enrichment, and response.

Overview
This project documents the build-out of a SOC automation pipeline using three open-source platforms:

Wazuh — SIEM and EDR
TheHive — Case management
Shuffle — SOAR (Security Orchestration, Automation and Response)

The lab simulates a real-world detection and response workflow: a Mimikatz execution on a Windows endpoint triggers a Wazuh alert, which is forwarded to Shuffle, enriched via VirusTotal, escalated to TheHive as a case, and notified to a SOC analyst by email, all automated.

Infrastructure
Component                     OS             Role
Windows Client            Windows 10        Endpoint with Wazuh Agent + Sysmon
Wazuh Server              Ubuntu 22.04      SIEM / EDR Manager
TheHive Server            Ubuntu 22.04      Case Management + Cassandra + Elasticsearch
Shuffle                   Cloud-hosted      SOAR Orchestration

