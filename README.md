# Microsoft-Sentinel-SIEM-Deployment-in-Azure
<img width="843" height="860" alt="Deploy Azure Sentinel drawio" src="https://github.com/user-attachments/assets/f6ed6935-35a5-4d4f-ab6d-e331a741c858" />

## Cloud SIEM Lab: Microsoft Sentinel with Windows 11 Honeypot

## Objective

This project demonstrates the deployment of a cloud-based Security Information and Event Management (SIEM) solution using Microsoft Azure. The objective is to set up Microsoft Sentinel (a modern, cloud-native SIEM) to collect and analyze security logs from a deliberately vulnerable Windows 11 virtual machine acting as a honeypot
mcfajao.com
. By exposing this VM to the internet (with RDP open and firewalls disabled), we attract brute-force login attacks and capture the activity in real time. Microsoft Sentinel – which integrates AI, SOAR (Security Orchestration, Automation, and Response), UEBA, and threat intelligence – enables us to detect and visualize these attacks across the cloud environment
microsoft.com
. This lab not only highlights the global and automated nature of cyberattacks, but also showcases skills in cloud infrastructure setup, log management, Kusto Query Language (KQL) analysis, and security monitoring. (In one example, an Azure honeypot faced over 5,000 RDP login attempts within 24 hours from attackers worldwide
medium.com
, underscoring the importance of such monitoring.)
Architecture

The diagram above illustrates the lab architecture. A Windows 11 VM in Azure (the honeypot) is configured to allow remote desktop attacks, and it continuously sends its Windows Event logs to an Azure Log Analytics Workspace via the Azure monitoring agent. Microsoft Sentinel is enabled on that workspace to aggregate and analyze the incoming log data as a cloud-native SIEM platform. The setup also includes a custom PowerShell log exporter on the VM that gathers attacker IP addresses from failed login events and uses a third-party geolocation API to enrich the data with location information
github.com
mcfajao.com
. The resulting log data (including coordinates of attack sources) is sent to Azure and visualized on a world map in Sentinel, providing a clear, real-time view of where attacks originate.
Components

    Azure Virtual Network (VNet) – Provides network connectivity for the VM (with a public IP for RDP access).

    Windows 11 Virtual Machine – Acts as the honeypot server. Configured with RDP (TCP 3389) open to the internet and local firewall disabled to entice attackers
    medium.com
    .

    Azure Network Security Group (NSG) – Controls inbound traffic to the VM. For this lab, the NSG allows global RDP access (temporarily, for honeypot purposes).

    Azure Log Analytics Workspace – Central log repository in Azure. All Windows security events from the VM are collected here (e.g. Windows Event ID 4625 for failed logon attempts)
    mcfajao.com
    .

    Microsoft Sentinel – Cloud-native SIEM front-end for the Log Analytics data. Used to query logs, set up detection rules, and create visualizations
    microsoft.com
    .

    Geolocation Service API – A third-party API (e.g. ipgeolocation.io or IP2Location) used by the VM’s script to translate attacker IP addresses into geographic locations (country, latitude/longitude)
    mcfajao.com
    .

    Azure Monitor Agent (AMA)/Log Analytics Agent – The agent installed on the VM to send logs to the Log Analytics Workspace. In this lab, it was enabled via Azure Defender for Cloud with all events collected
    mcfajao.com
    .

High-Level Steps

    Deploy Azure VM – Create an Azure resource group and a Windows 11 VM for the honeypot. Configure networking to allow RDP from anywhere and disable the VM’s firewall to simulate an intentionally vulnerable target
    medium.com
    .

    Configure Log Collection – Create a Log Analytics Workspace and link the VM to it. Enable collection of Windows Event logs (Security events) from the VM so that all login attempts and other events are reported to Azure
    mcfajao.com
    mcfajao.com
    .

    Enable Microsoft Sentinel – Turn on Azure Sentinel by attaching it to the Log Analytics Workspace. This sets up the SIEM environment, allowing us to use Sentinel’s tools to query and visualize the ingested logs
    mcfajao.com
    .

    Simulate Attacks – Generate failed login attempts on the VM (e.g. incorrect RDP login) to verify logs are flowing. Then leave the VM running with exposed RDP to attract real brute-force attacks from the internet. The incoming attack data (failed login events) will be captured in Azure Sentinel
    medium.com
    .

    Deploy Geolocation Script – Run a PowerShell script on the VM that monitors the Security event log for failed logins, extracts the source IPs, and queries a geolocation API. The script continuously writes out a custom log file (failed_rdp.log) with each attack attempt’s timestamp, source IP, and location (latitude, longitude, country, etc.)
    github.com
    mcfajao.com
    .

    Ingest Custom Logs – Configure Azure Monitor to ingest the custom log file from the VM. Using the Log Analytics workspace settings, create a Custom Log that watches the path of the failed_rdp.log on the VM and transmits new entries to Azure
    mcfajao.com
    mcfajao.com
    . Verify that the new custom log data (e.g. Failed_RDP_With_Geo table) is appearing in Sentinel.

    Visualize Attacks on Map – In Microsoft Sentinel, create a new Workbook to display the honeypot attack data. Use KQL queries to parse the custom log (extracting fields like username, source IP, and location) and then add a Map visualization
    mcfajao.com
    mcfajao.com
    . Configure the map to use latitude/longitude from the log data and to show attack counts (heatmap or point markers) per location. This live dashboard will update as new attacks occur, plotting the origin of each failed RDP attempt on a world map
    mcfajao.com
    .

Below, we document the implementation steps in detail:
Step 1: Deploy the Windows 11 Honeypot VM

First, set up the Azure environment and vulnerable VM. Using the Azure Portal, create a Resource Group for the lab and then create a new Windows 11 Virtual Machine (use a Windows 10 image if Windows 11 is not available, as the process is similar). While configuring the VM, choose an appropriate size (e.g., Standard D2s_v3) and ensure RDP (TCP 3389) is allowed in the inbound port rules during creation
mcfajao.com
mcfajao.com
. For convenience, deploy the VM in the same region as your Log Analytics workspace will be. Set an administrative username/password for the VM (and remember these for RDP access).

Once the VM is running, adjust its security settings to function as a honeypot. Disable the Windows Firewall on all profiles (Domain, Private, Public) so that the VM does not block any incoming connections
mcfajao.com
medium.com
. This can be done by RDP-ing into the VM and using the Windows Defender Firewall management console (wf.msc) to turn off the firewall for all network profiles. By removing this layer of defense and opening RDP to the world, the VM becomes an attractive target for automated attack bots scanning the internet. (This is done for learning purposes only – in a real environment, such exposure is dangerous. As a safety note, Azure may charge minimal usage fees, and it’s good practice to delete these resources when finished to avoid ongoing costs.)
Step 2: Create Log Analytics Workspace and Enable Data Collection

Next, set up a Log Analytics Workspace in Azure, which will collect logs from the VM. In the Azure Portal, search for Log Analytics Workspaces and create a new workspace. Use the same resource group as the VM and select the region closest to your VM for optimal performance
mcfajao.com
. Once deployed, this workspace will act as the cloud database for our logs.

After creating the workspace, configure the VM to send its logs to it. Azure offers an easy way to onboard VM logs via Microsoft Defender for Cloud (formerly Azure Security Center). Navigate to Microsoft Defender for Cloud > Environment Settings, then select your subscription and the resource (the VM or its resource group). Enable the Defender for Servers plan if prompted (this installs the Azure Monitoring Agent). Under the data collection settings, choose to Collect “All Events” from Windows event logs (especially Security logs)
mcfajao.com
. This setting ensures that every Windows event (all levels, e.g. Information, Warning, Error) will be forwarded. Save these settings.

Now link the VM to the Log Analytics workspace. In the Defender for Cloud portal or directly in the Log Analytics Workspace settings, find the Virtual Machines list and locate your new VM. There should be an option to Connect the VM to the workspace
mcfajao.com
. Clicking "Connect" will deploy the log analytics agent (if not already present) and start the data ingestion. After a few minutes, the VM will send a heartbeat to confirm connectivity, and its Windows events will begin flowing to Azure. You can verify the connection in the Log Analytics workspace under Agents management or by running a simple log query for the VM’s heartbeat. At this point, Azure is actively collecting logs from the Windows 11 honeypot.
Step 3: Enable Microsoft Sentinel on the Workspace

With the workspace receiving data, enable Microsoft Sentinel to leverage SIEM capabilities. In the Azure Portal, search for Microsoft Sentinel and open its interface. Click Add and select the Log Analytics workspace you created for this project, then click Add Microsoft Sentinel
mcfajao.com
. This attaches Sentinel to that workspace (it might take a moment to provision). Once done, the workspace is now “SIEM-enabled”.

In the Sentinel overview, you’ll see options for creating analytics rules, workbooks, hunting queries, etc. But first, let's confirm that raw logs are visible in Sentinel. Navigate to Logs (under the Sentinel or the workspace blade) and run a basic query on the SecurityEvent table (this table contains Windows Event logs by default). For example, execute a query to find failed login events:

SecurityEvent
| where EventID == 4625
| take 5

Event ID 4625 corresponds to Windows login failures. If everything is configured correctly, you should see results for the test failed login (from when you intentionally entered a wrong password) or any attacks that have already started
mcfajao.com
. This confirms that Microsoft Sentinel is ingesting the VM’s security logs. The SIEM is now operational, and you can proceed to set up advanced visualization and analysis.
Step 4: Simulate and Observe Malicious Login Attempts

With the honeypot in place and logging enabled, the next step is to generate and observe attack data. Start by performing a manual test: attempt to log into the VM via RDP with incorrect credentials. For example, use the correct username but an incorrect password on the first try (then the correct password on the second try to actually get in). This will create a failed logon event (4625) followed by a successful one (4624) in the Windows Security log, which will be sent to Sentinel
github.com
. You can check the VM’s Event Viewer (Security log) to see the events locally, and then verify they appear in Azure Sentinel’s log query results as well. Seeing your test attempt in Sentinel confirms end-to-end data flow.

Now, the real honeypot experiment begins. Leave the VM running continuously with RDP accessible to the internet (and no firewall). It may take some time, but malicious actors on the internet will eventually discover the open RDP port and begin brute-force attacks, trying common usernames and passwords. These unauthorized login attempts will all generate security events on the VM, which are forwarded to Sentinel. You might start seeing a trickle of 4625 events from unfamiliar source IP addresses, and over hours this can escalate dramatically. (As noted in a similar Azure honeypot, over 5,000 failed login attempts can hit within the first day
medium.com
!) The attacks often come from many different IPs across the world, illustrating the global nature of brute-force threats. All these events are being recorded in your Log Analytics workspace for analysis.

    Note: Ensure the VM’s NSG allows the traffic (the default RDP rule for “Any” source on port 3389 should suffice). For capturing ICMP/ping or other types of traffic, you could open additional ports or use custom honeypot services, but RDP is usually enough to attract password-guessing bots. Also, keep an eye on Azure costs; the log data volume from a day of attacks is not large (a few thousand events), but the VM running continuously does incur some compute charges (which are minimal for a single small VM over a short period). 

Step 5: Deploy Geolocation Script on the VM

Collecting raw event logs is informative, but the data becomes far more insightful when enriched with context like the attacker’s geographic location. To achieve this, we deploy a geolocation lookup script on the Windows VM. This is a PowerShell script that will continuously monitor the Windows Security log for new failed login events and, for each one, call an external API to get the approximate location of the source IP address
github.com
. The location (country, city, latitude, longitude, etc.) is then recorded along with the event details.

Prepare the API: Sign up for a free account with an IP geolocation service (for example, ipgeolocation.io which offers ~1000 free lookups, or IP2Location which offers a larger trial quota)
mcfajao.com
. Obtain an API key for the service – this key will be used by the script to query the geolocation database.

Run the PowerShell script: On the honeypot VM, open PowerShell ISE (or Notepad) with administrative privileges and create a new script file (e.g., GeoLocateFailedLogins.ps1). Paste in the custom script code (provided by the lab or the reference project) and insert your API key in the designated variable field
mcfajao.com
. The script typically works as follows: it uses Windows Event Log queries to find any new event ID 4625 entries (failed logons), extracts the source IP address from each event, sends a query to the geolocation API for that IP, then writes a line to a log file with the relevant fields (timestamp, source IP, country, latitude, longitude, etc.). Save the script and execute it on the VM.

Let the script run continuously. Upon each failed login event, it will append an entry to a local log file. By default, in our setup, we name this file C:\ProgramData\failed_rdp.log. You can open this file to verify that it’s being populated with records as attacks occur (each line might be a CSV or JSON-like line containing the data for one attempt). For example, an entry might include something like timestamp, sourcehost (IP), country, state, latitude, longitude, username and so on, representing one failed RDP login and its origin details. Running the script effectively creates a custom telemetry stream of enriched security events on the VM
mcfajao.com
.
Step 6: Ingest the Custom Geolocation Log into Azure Sentinel

Now that the VM is producing a custom log file with geolocation info, we need Azure to ingest this data so we can analyze it in Sentinel. Azure Monitor can collect custom logs via the Log Analytics agent (MMA) since we enabled that earlier. We’ll configure the workspace to pick up failed_rdp.log from the VM.

Go to your Log Analytics Workspace in the Azure Portal. Navigate to Agents management or Custom Logs (in the workspace settings). Click Add Custom Log. In the wizard:

    Upload a sample of the log file (download a copy of failed_rdp.log from the VM or copy some recent lines into a text file)
    mcfajao.com
    . This allows Azure to parse the structure. Use the default line breaks as record delimiters.

    Specify the log collection path on the target machines. In our case, we enter the Windows path where the log is stored, for example: C:\ProgramData\failed_rdp.log (or the directory C:\ProgramData\ with a wildcard). This path tells the agent which file to monitor. Ensure the path is correct and accessible – the Azure VM’s agent will start reading this file
    mcfajao.com
    .

    Provide a name for the custom log type, such as Failed_RDP_With_GEO. Azure will append “_CL” to this name to form the table name in Log Analytics (e.g., Failed_RDP_With_GEO_CL).

    Complete the wizard to create the custom log. It may take some minutes for the new configuration to propagate. Once set up, the agent will begin sending the contents of failed_rdp.log to the Log Analytics Workspace. (Only new log entries will be ingested going forward; the historical lines you uploaded as sample might also appear for reference.)

After waiting ~10-15 minutes
mcfajao.com
, verify that the custom log data is flowing. In the Log Analytics Logs query interface, look for a new table with the name you chose (it should end in _CL). For example, run:

Failed_RDP_With_GEO_CL
| take 5

You should see records containing the raw text of each log entry (often in a field called RawData or similar). Each record corresponds to a failed login attempt captured by the script, including the geolocation info. If you see data here, congratulations – your custom log pipeline from the VM to Azure is working.
Step 7: Create a World Map Visualization in Sentinel

With the geolocation-enriched data available in Sentinel, the final step is to create a visual representation of the attacks. We will build a Workbook in Microsoft Sentinel that plots the failed RDP attempts on a world map.

Go to the Microsoft Sentinel interface and select Workbooks from the menu. Click on + Add workbook to create a new blank workbook. In the workbook editor, add a new query tile: choose Add query and select the Log Analytics workspace as the data source. This allows us to query our custom log within the workbook.

Now, write a KQL query that retrieves the relevant fields from the custom log. The raw log text includes embedded fields like username, source IP, country, latitude, longitude, etc. We can use the Kusto extract() function or the UI-based field extractor to pull these out into separate columns. For example, a KQL query might look like (adjust the custom log table name accordingly):

Failed_RDP_With_GEO_CL
| extend Timestamp = extract(@"timestamp:([^,]+)", 1, RawData)
| extend SourceIP = extract(@"sourcehost:([^,]+)", 1, RawData)
| extend Country = extract(@"country:([^,]+)", 1, RawData)
| extend Latitude = extract(@"latitude:([^,]+)", 1, RawData)
| extend Longitude = extract(@"longitude:([^,]+)", 1, RawData)
| summarize Count = count() by Country, Latitude, Longitude, SourceIP

This query takes each log entry (RawData) and uses regex patterns to pull out the timestamp, source IP, country, latitude, and longitude values
mcfajao.com
. It then aggregates (summarize) the data by unique location (we group by country and coordinates, and maybe source IP) to count how many attempts came from each. You can include other fields like username or destination host if needed. (Alternatively, Azure Sentinel’s GUI offers an Extract Fields feature where you highlight text in a sample log entry to create custom fields without writing regex – either approach achieves the goal of structuring the data
github.com
.)

Run the query in the workbook editor to ensure it returns results (you should see each attacking IP or location with a count of attempts). Next, click on the Visualization options for that query and switch the visual from a table to a Map. Configure the map settings to use the latitude and longitude fields for plotting points. For example, set Location to Latitude/Longitude, assigning Latitude to the latitude field and Longitude to the longitude field from your query results
mcfajao.com
. You can choose a map style such as heatmap or clustered points. For a heatmap effect, you might use the Count of attempts as the intensity – in the settings, set Size or Color by to the Count (number of attempts) and choose a gradient color palette (e.g., green-to-red) to represent low-to-high attack volume
mcfajao.com
. Also, you can add a label to each point showing the country or IP (for instance, use Country as the label).

After configuring, the workbook will display a world map with dots or heat spots indicating the origin of each failed RDP attack on your honeypot. Each time a new attack is logged and the custom log updates, the map can be refreshed (you can set an auto-refresh interval) to show the latest data
mcfajao.com
. Save the workbook (give it a name like "Honeypot Attack Map") and pin it to your Sentinel dashboard if desired.

At this stage, you have a live security operations dashboard: as attackers from around the globe attempt to breach the RDP login, their attempts are being logged, sent to the cloud, enriched with geolocation, and visualized in real-time. This fulfills the lab's goal of making the data easy to understand at a glance
mcfajao.com
.
Results and Analysis

By completing this project, we stood up a mini SOC (Security Operations Center) in the cloud, capable of capturing and analyzing real attack data. The world map visualization in Sentinel provides an intuitive view of the origin of threats: for example, you might observe clusters of attacks from specific countries or regions that are notorious for botnet activity. In one observed run of this lab, a single attacker in South America generated over 2,900 RDP login attempts within a 55-minute window
mcfajao.com
, while another experiment recorded 5,000+ attempts from diverse locations in 24 hours
medium.com
. This reinforces how quickly and relentlessly exposed systems can be targeted on the internet.

From a skills perspective, this lab showcases proficiency in several areas: cloud infrastructure deployment, security monitoring, log analytics, and incident visualization. We used Azure services to build an end-to-end pipeline – from configuring a VM and network settings, to enabling Azure Sentinel and Defender for Cloud, to writing KQL queries and custom scripts for data enrichment. This kind of project demonstrates the ability to not only set up complex cloud environments but also derive meaningful security insights from raw data. For potential employers, the project highlights hands-on experience with Microsoft Azure and Sentinel, an understanding of SIEM concepts, and an appreciation for proactive cybersecurity measures.

Next Steps: In a real-world scenario, one would follow up by implementing defenses for the vulnerabilities identified. For instance, after observing the attack patterns, we could enable firewall rules or Azure Sentinel analytics rules to alert on brute-force behavior. We would also ensure the honeypot is safely isolated. Nonetheless, the knowledge gained from this lab is invaluable for understanding attacker behavior and sharpening skills in cloud security operations.
