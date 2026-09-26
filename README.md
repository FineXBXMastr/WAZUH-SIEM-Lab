# WAZUH-SIEM-Lab

## Overview

This repository documents a self-hosted Active Directory and SIEM lab built to 
gain hands-on, practical experience with the kind of environment a security 
analyst or detection engineer works in day to day. The lab was built entirely 
on a single laptop using VMware Workstation Pro, simulating a small enterprise 
network complete with a domain controller, a domain-joined client, an attacker 
system, a segmented firewall, and a SIEM for log ingestion and alerting.

Rather than relying on paid cloud platforms, this project was built with free 
and open-source tooling to demonstrate that the core skills of detection 
engineering — attack simulation, log pipeline configuration, and alert 
triage — can be practiced end to end without significant cost.

## What this project demonstrates

- Standing up Active Directory Domain Services from scratch, including domain 
  controller promotion, DNS configuration, and domain user management
- Joining and managing a Windows client within a domain environment
- Deploying an open-source SIEM (Wazuh) and configuring agent-based log 
  forwarding from Windows hosts
- Simulating realistic attacker behavior (credential brute-forcing, privilege 
  escalation) using industry-standard offensive tooling
- Verifying that simulated attacks are visible and detectable within the SIEM, 
  using both default detection rules and custom-authored correlation rules
- Segmenting the attacker from the protected environment using a pfSense 
  firewall, including narrow, least-privilege rule authoring to selectively 
  re-enable specific attack paths for controlled testing
- Troubleshooting real infrastructure issues encountered along the way — 
  virtual networking conflicts, service failures, IP address conflicts, and 
  decoder/field-mapping gaps in SIEM rule logic — which reflects the kind of 
  hands-on problem-solving this work actually involves

## Documentation

- [Lab Setup](docs/lab-setup.md) — full breakdown of each virtual machine, its 
  configuration, and its role in the lab
- [Attack Chains](docs/attack-chains.md) — documented attack scenarios, the 
  steps taken to execute them, and the resulting detections observed in Wazuh
- [Firewall Addition](docs/firewall-addition.md) — introduction of a pfSense 
  firewall to segment the attacker from the protected environment, and the 
  resulting default-deny behavior observed
- [Custom Rules](docs/custom-rules.md) — authoring a custom Wazuh correlation 
  rule for brute-force detection, the firewall changes required to re-test it, 
  and the troubleshooting involved in getting it to fire correctly

## Environment summary

| Role | OS | Static IP |
|---|---|---|
| Domain Controller | Windows Server 2022 Standard | `10.10.5.10` |
| Domain Client | Windows 10 Enterprise | `10.10.5.11` |
| SIEM (Wazuh Manager) | Ubuntu Server 24.04 | `10.10.5.20` |
| Firewall (pfSense) — LAN | pfSense CE 2.9.0 | `10.10.5.2` |
| Firewall (pfSense) — WAN | pfSense CE 2.9.0 | `10.10.6.2` |
| Attacker | Kali Linux | `10.10.6.12` |

All virtual machines run on VMware Workstation Pro. The protected environment 
(domain controller, client, and SIEM) sits on an isolated host-only network 
segment, separate from both the host machine's home network and the attacker's 
segment, with pfSense enforcing and logging traffic between the two.

## Future work

Planned additions to this lab include further attack scenarios (lateral 
movement, credential access via MITRE ATT&CK-mapped Atomic Red Team tests), 
network-based intrusion detection via a pfSense package (Suricata or Snort), 
and additional network segmentation to separate the domain controller and 
client onto distinct subnets for more granular firewall rule enforcement.