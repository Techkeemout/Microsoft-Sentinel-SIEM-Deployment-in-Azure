# Microsoft-Sentinel-SIEM-Deployment-in-Azure

## Cloud SIEM Lab: Microsoft Sentinel with Windows 11 Honeypot

## Objective

 This project demonstrates the deployment of a cloud-based Security Information and Event Management (SIEM) solution using Microsoft Azure. The objective is to set up Microsoft Sentinel (a modern, cloud-native SIEM) to collect and analyze security logs from a deliberately vulnerable Windows 11 virtual machine acting as a honeypot
By exposing this VM to the internet (with RDP open and firewalls disabled), we attract brute-force login attacks and capture the activity in real time. Microsoft Sentinel – which integrates AI, SOAR (Security Orchestration, Automation, and Response), UEBA, and threat intelligence – enables us to detect and visualize these attacks across the cloud environment This lab not only highlights the global and automated nature of cyberattacks, but also showcases skills in cloud infrastructure setup, log management, Kusto Query Language (KQL) analysis, and security monitoring. 

## Architecture

<img width="843" height="986" alt="Deploy Azure Sentinel drawio" src="https://github.com/user-attachments/assets/6ccfb33f-43b6-422f-bbf1-8b1b7874ab68" />


 The diagram above illustrates the lab architecture. A Windows 11 VM in Azure (the honeypot) is configured to allow remote desktop attacks, and it continuously sends its Windows Event logs to an Azure Log Analytics Workspace via the Azure monitoring agent. Microsoft Sentinel is enabled on that workspace to aggregate and analyze the incoming log data as a cloud-native SIEM platform.
The resulting log data (including coordinates of attack sources) is sent to Azure and visualized on a world map in Sentinel, providing a clear, real-time view of where attacks originate.

## Components

- Azure Virtual Network (VNet) – Provides network connectivity for the VM (with a public IP for RDP access).
- Windows 11 Virtual Machine – Acts as the honeypot server. Configured with RDP (TCP 3389) open to the internet and local firewall disabled to entice attackers.
- Azure Network Security Group (NSG) – Controls inbound traffic to the VM. For this lab, the NSG allows global RDP access (temporarily, for honeypot purposes).
- Azure Log Analytics Workspace – Central log repository in Azure. All Windows security events from the VM are collected here (e.g. Windows Event ID 4625 for failed logon attempts)
- Microsoft Sentinel – Cloud-native SIEM front-end for the Log Analytics data. Used to query logs, set up detection rules, and create visualizations.
- Geolocation Service API – A third-party API (e.g. ipgeolocation.io or IP2Location) used by the VM’s script to translate attacker IP addresses into geographic locations (country, latitude/longitude)
- Azure Monitor Agent (AMA)/Log Analytics Agent – The agent installed on the VM to send logs to the Log Analytics Workspace. In this lab, it was enabled via Azure Defender for Cloud with all events collected.

## High-Level Steps

1. Deploy Azure VM – Create an Azure resource group and a Windows 11 VM for the honeypot. Configure networking to allow RDP from anywhere and disable the VM’s firewall to simulate an intentionally vulnerable target

2. Configure Log Collection – Create a Log Analytics Workspace and link the VM to it. Enable collection of Windows Event logs (Security events) from the VM so that all login attempts and other events are reported to Azure.

3. Enable Microsoft Sentinel – Turn on Azure Sentinel by attaching it to the Log Analytics Workspace. This sets up the SIEM environment, allowing us to use Sentinel’s tools to query and visualize the ingested logs.

4. Simulate Attacks – Generate failed login attempts on the VM (e.g. incorrect RDP login) to verify logs are flowing. Then leave the VM running with exposed RDP to attract real brute-force attacks from the internet. The incoming attack data (failed login events) will be captured in Azure Sentinel medium.com.

5. Deploy Geolocation Script – Run a PowerShell script on the VM that monitors the Security event log for failed logins, extracts the source IPs, and queries a geolocation API. The script continuously writes out a custom log file (failed_rdp.log) with each attack attempt’s timestamp, source IP, and location (latitude, longitude, country, etc.) github.com.

6. Ingest Custom Logs – Configure Azure Monitor to ingest the custom log file from the VM. Using the Log Analytics workspace settings, create a Custom Log that watches the path of the failed_rdp.log on the VM and transmits new entries to Azure. Verify that the new custom log data (e.g. Failed_RDP_With_Geo table) is appearing in Sentinel.

7. Visualize Attacks on Map – In Microsoft Sentinel, create a new Workbook to display the honeypot attack data. Use KQL queries to parse the custom log (extracting fields like username, source IP, and location) and then add a Map visualization. Configure the map to use latitude/longitude from the log data and to show attack counts (heatmap or point markers) per location. This live dashboard will update as new attacks occur, plotting the origin of each failed RDP attempt on a world map.

Below, I  document the implementation steps in detail:

## Step 1: Deploy the Windows 11 Honeypot VM

First, set up the Azure environment and vulnerable VM. Using the Azure Portal, create a Resource Group for the lab.  

<img width="1454" height="731" alt="Create Resource Group" src="https://github.com/user-attachments/assets/af8a02ba-bb44-4cbb-8565-e849e70379dc" />
<img width="1676" height="768" alt="image" src="https://github.com/user-attachments/assets/3b106490-d33d-4fe8-b3cd-c920ef0e8ac1" />

- Next, create a new Windows 11 Virtual Machine (use a Windows 10 image if Windows 11 is not available, as the process is similar).  
<img width="2054" height="1232" alt="Create VM" src="https://github.com/user-attachments/assets/39426ca3-8c9e-4a6d-9d7d-f9d6bd6487e6" />

- For convenience, deploy the VM in the same region as your Log Analytics workspace will be. 
<img width="1307" height="1215" alt="Create VM Basics " src="https://github.com/user-attachments/assets/3c09a5ac-5df0-41a8-a95c-634cef0c3506" />

- Ensure RDP (TCP 3389) is allowed in the inbound port rules during creation. Set an administrative username/password for the VM (and remember these for RDP access).
<img width="2004" height="1008" alt="Create VM Allow RDP" src="https://github.com/user-attachments/assets/0a239420-b1d3-4d0f-8742-368c233b8255" />

Once the VM is running, adjust its security settings to function as a honeypot. Disable the Windows Firewall on all profiles (Domain, Private, Public) so that the VM does not block any incoming connections.
medium.com. This can be done by RDP-ing into the VM and using the Windows Defender Firewall management console (wf.msc) to turn off the firewall for all network profiles. By removing this layer of defense and opening RDP to the world, the VM becomes an attractive target for automated attack bots scanning the internet. (This is done for learning purposes only – in a real environment, such exposure is dangerous. As a safety note, Azure may charge minimal usage fees, and it’s good practice to delete these resources when finished to avoid ongoing costs.)

## Step 2: Create Log Analytics Workspace and Enable Data Collection

Next, set up a Log Analytics Workspace in Azure, which will collect logs from the VM. In the Azure Portal, search for **Log Analytics Workspaces** and create a new workspace. Use the same resource group as the VM and select the region closest to your VM for optimal performance. Once deployed, this workspace will act as the cloud database for our logs.

<img width="1456" height="884" alt="Log analytics workspace" src="https://github.com/user-attachments/assets/003d9da7-ab47-4777-ae0a-55194d3c8d3d" />



After creating the workspace, configure the VM to send its logs to it. Azure offers an easy way to onboard VM logs via Microsoft Defender for Cloud (formerly Azure Security Center). Navigate to Microsoft Defender for Cloud > Environment Settings, then select your subscription and the resource (the VM or its resource group). Enable the Defender for Servers plan if prompted (this installs the Azure Monitoring Agent). Under the data collection settings, choose to Collect “All Events” from Windows event logs (especially Security logs). This setting ensures that every Windows event (all levels, e.g. Information, Warning, Error) will be forwarded. Save these settings.

Now link the VM to the Log Analytics workspace. In the Defender for Cloud portal or directly in the Log Analytics Workspace settings, find the Virtual Machines list and locate your new VM. There should be an option to Connect the VM to the workspace. Clicking "Connect" will deploy the log analytics agent (if not already present) and start the data ingestion. After a few minutes, the VM will send a heartbeat to confirm connectivity, and its Windows events will begin flowing to Azure. You can verify the connection in the Log Analytics workspace under Agents management or by running a simple log query for the VM’s heartbeat. At this point, Azure is actively collecting logs from the Windows 11 honeypot.

## Step 3: Enable Microsoft Sentinel on the Workspace
<img width="1829" height="1317" alt="Microsoft Sentinel" src="https://github.com/user-attachments/assets/ed689711-e6ae-40bd-8957-56e6787ff594" />

With the workspace receiving data, enable Microsoft Sentinel to leverage SIEM capabilities. In the Azure Portal, search for Microsoft Sentinel and open its interface. 

Click Add and select the Log Analytics workspace you created for this project, then click Add Microsoft Sentinel. This attaches Sentinel to that workspace (it might take a moment to provision). Once done, the workspace is now “SIEM-enabled”.

In the Sentinel overview, you’ll see options for creating analytics rules, workbooks, hunting queries, etc. But first, let's confirm that raw logs are visible in Sentinel. Navigate to Logs (under the Sentinel or the workspace blade) and run a basic query on the SecurityEvent table (this table contains Windows Event logs by default). For example, execute a query to find failed login events:

SecurityEvent
| where EventID == 4625
| take 5

Event ID 4625 corresponds to Windows login failures. If everything is configured correctly, you should see results for the test failed login (from when you intentionally entered a wrong password) or any attacks that have already started This confirms that Microsoft Sentinel is ingesting the VM’s security logs. The SIEM is now operational, and you can proceed to set up advanced visualization and analysis.

## Step 4: Simulate and Observe Malicious Login Attempts

With the honeypot in place and logging enabled, the next step is to generate and observe attack data. Start by performing a manual test: attempt to log into the VM via RDP with incorrect credentials. For example, use the correct username but an incorrect password on the first try (then the correct password on the second try to actually get in). This will create a failed logon event (4625) followed by a successful one (4624) in the Windows Security log, which will be sent to Sentinel github.com. You can check the VM’s Event Viewer (Security log) to see the events locally, and then verify they appear in Azure Sentinel’s log query results as well. Seeing your test attempt in Sentinel confirms end-to-end data flow.

Now, the real honeypot experiment begins. Leave the VM running continuously with RDP accessible to the internet (and no firewall). It may take some time, but malicious actors on the internet will eventually discover the open RDP port and begin brute-force attacks, trying common usernames and passwords. These unauthorized login attempts will all generate security events on the VM, which are forwarded to Sentinel. You might start seeing a trickle of 4625 events from unfamiliar source IP addresses, and over hours this can escalate dramatically. The attacks often come from many different IPs across the world, illustrating the global nature of brute-force threats. All these events are being recorded in your Log Analytics workspace for analysis.

Note: Ensure the VM’s NSG allows the traffic (the default RDP rule for “Any” source on port 3389 should suffice). For capturing ICMP/ping or other types of traffic, you could open additional ports or use custom honeypot services, but RDP is usually enough to attract password-guessing bots. Also, keep an eye on Azure costs; the log data volume from a day of attacks is not large (a few thousand events), but the VM running continuously does incur some compute charges (which are minimal for a single small VM over a short period). 

## This project is still ongoing...
