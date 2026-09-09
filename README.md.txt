My goal for my Wazuh SIEM Lab is to create a cybersecurity environment that might be observed in a real enterprise or business. My plan was to gain experience with a real SIEM by employing my existing system administration, virtualization, and penetration testing skills as well as new skills to create this environment and then initiate attack chains that can be monitored in the SIEM. To do so, I have configured four virtual machines with VMware to operate on their own virtual network. The virtual machines are as follows:

Windows Server 2022 - Acts as the domain controller of the cyber.local domain
Windows 10 - A client machine under the jurisdiction of the domain controller. Accounts configured through Active Directory on the DC will login on this machine
Ubuntu Server 26.04 - The machine responsible for the Wazuh service. Wazuh is hosted through this vm and can be accessed via my web browser
Kali Linux - A machine to be used as an attacker. RDP brute force and rouge admin account creation, as well as other attack chains, will be done through here

The four VMs are all on the same subnet, that being 10.10.5.0/24. The IP addresses are as follows:

Windows Server 2022 - 10.10.5.10
Windows 10 - 10.10.5.11
Ubuntu Server - 10.10.5.20
Kali - 10.10.5.12 (in a real environment, Kali, as an external attacker machine, would be on a different subnet)

Details for the setup of the lab, including DC promotion, AD administration, Wazuh setup, agent deployment, and IP configuration are included in this repository. Attack chain details are included as well.