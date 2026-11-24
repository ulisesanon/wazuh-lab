# wazuh-lab
Home SOC lab: Wazuh SIEM monitoring Linux and Windows endpoints on an isolated virtual network, with simulated attacks and incident reports.

This is a home security operations lab I built to practice the Tier 1 SOC workflow: get logs flowing, run attacks, catch them, and write them up.

Everything runs on a MacBook Pro (Apple Silicon) in UTM virtual machines on an isolated shared network. A Wazuh 4.14 server (indexer, manager and dashboard on one Ubuntu Server VM) collects logs from two endpoints: an Ubuntu Server machine standing in for a school's Linux server, and a Windows 11 Pro workstation. Each endpoint runs a Wazuh agent and reports back to the server.

Machine	Role	IP
  
	wazuh	SIEM server [indexer, manager, dashboard]	
    192.168.64.100
  
	ubuntu-endpoint	[Monitored Linux server]
    192.168.64.101
  
	win11-endpoint [Monitored Windows workstation]	
    192.168.64.102

Status: server and both endpoints are deployed and reporting. Next up is a Kali attacker VM and three simulated attacks (SSH brute force, a backdoor admin account, and tampering with fake student records), each with an incident report mapped to MITRE ATT&CK.
