# Deploy BIG-IP on GCP with GDM without Internet access

## Contents

- [Deploy BIG-IP on GCP with GDM without Internet access](#deploy-big-ip-on-gcp-with-gdm-without-internet-access)
  - [Contents](#contents)
  - [Introduction](#introduction)
  - [Solution Description](#solution-description)
    - [Network](#network)
    - [DNS](#dns)
    - [Google Cloud Storage](#google-cloud-storage)
    - [BIG-IP GDM template](#big-ip-gdm-template)
  - [Testing](#testing)
  - [Conclusion](#conclusion)
  - [Troubleshooting](#troubleshooting)
  - [References](#references)

(*Special thanks to Matthew Emes, F5 Global SA, for being my shadow to get this solution to work, supporting me in adding the gcURLs script and show me some decent troubleshooting. Read his article about isolated BIG-IPs and CFE in GCP using Terraform: https://github.com/memes/f5-google-bigip-isolated-vpcs)*


## Introduction
Last week I got involved in a customer use cases where the challenge was to deploy F5 BIG-IP in Google Cloud Platform by using Google Deployment Manager. Yes, using our F5 Cloud Solutions templates to deploy F5 BIG-IP into GCP.

What in that case is easier than grab the template of choice fill in this template and do a “gcloud deployments-manager create…” to deploy though GCP CLI and before you know it, your deployment is up and running, ready to be used.
But no, not this time, since the security requirements for this customer are that there is no Internet access which can be used at runtime to get our cloud-libs and assorted packages. 

“Houston, we have problem!”.

![](png/diagram.png)

## Solution Description

The only way this solution can work is where you are able to download the F5 packages from a GCP local storage like Google Cloud Storage (GCS). This implies that the GDM template must have defined how to should reach out to this GCS to get to the packages and this can only be done from within the template definition through the use of a service account with the right authority. Another point to recon with is API traffic, normally the GDM template reaches out to *.googleapis.com and no Internet means that this needs to get resolved different and finally the VPC networks connecting the BIG-IP VM need to have access to the API and be able to reach Google API services.

Google Cloud Platform has the mentioend requirements build in and the following tasks need to be accomplished:

 - Enable restricted API access on subnets.
 - Create customer routes for the google API network 199.36.153.4/30
 - Create an internal DNS zone which overrides entries for *.googleapis.com
 - A GCS bucket with attached service account and SSH keys allowing retrieving locally stored F5 cloud-libs, GCE cloud-libs and F5 Automation Toolchain files.
 - A tweaked F5 BIG-IP GDM template. 
The Standalone-3NIC GDM template its original existence can found here: https://github.com/F5Networks/f5-google-gdm-templates/tree/master/supported/standalone/3nic/existing-stack/payg.

Quite a list so let’s started straight away.

### Network

**Private Google Access** is used to allow VMs without an external IP address to be able to send packets to external IP addresses used by Google APIs and services. For getting the isolated use case to work, we need all our three pre-defined subnets: external, management and internal to have Private Google Access enabled.

![](png/private-google-access.png)

This makes that each subnet is able to reach the GCS.
Now we are already in the subnet section, let’s create the custom route as well.

![](png/google-api-route.png)

This custom route will have a destination to the external addresses of Google API whose next hop is the default Internet gateway. Traffic send to Google APIs 199.36.153.4/30 will remain within Google’s network.

### DNS

Next, we need to create the **private DNS zone**. 
I have noticed that the GCP GUI might through you an error when creating the CNAME record and ‘restricted’ A records. So, let’s switch to gcloud and complete this task.
Setting up a private DNS zone is needed to have each VM in the VPC network be able to resolve *.googleapis.com as a CNAME to restricted.googleapis.com and configure an A record for restricted.googleapis.com.
The first step is to create the private DNS zone and add the VPC networks.

```
gcloud beta dns managed-zones create googleapis \
 --visibility=private \
 --networks=https://www.googleapis.com/compute/v1/projects/f5-gcs-4261-sales-emea-sa/global/networks/gert-vpc-external, https://www.googleapis.com/compute/v1/projects/f5-gcs-4261-sales-emea-sa/global/networks/gert-vpc-management, https://www.googleapis.com/compute/v1/projects/f5-gcs-4261-sales-emea-sa/global/networks/gert-vpc-internal\
 --dns-name=googleapis.com
```

Next, start a transaction.
```
gcloud dns record-sets transaction start --zone=googleapis
```
Now, create the DNS CNAME record which allows resolving *.googleapis.com to restricted.googleapis.com
```
gcloud dns record-sets transaction add --name=*.googleapis.com. \
    --type=CNAME restricted.googleapis.com. \
    --zone=googleapis \
    --ttl=300
```
And add the DNS A record resolving restricted.googleapis.com to the Google API external IP addresses.
```
gcloud dns record-sets transaction add --name=restricted.googleapis.com. \
    --type=A 199.36.153.4 199.36.153.5 199.36.153.6 199.36.153.7 \
    --zone=googleapis \
    --ttl=300
```
Finally, execute the transaction.
```
gcloud dns record-sets transaction execute --zone=googleapis
```
### Google Cloud Storage

Now the network and DNS piece are ready, it is time to create a GCS bucket. 

![](png/gcs.png)

The GCS bucket is used to store the cloud-libs and F5 automation toolchain packages. In this example f5-cloud-libs-gce, f5-cloud-libs and the AS3 .rpm are downloaded and added to the bucket.
Be sure to add a service account to the defined bucket which is allowed to grab the files from GCS.

### BIG-IP GDM template

The final piece is to customize the GDM template and this was the hardest piece to accomplish.
The used template is f5-existing-stack-payg-3nic-bigip though the customer use case requested BYOL. But using PAYG is just handy for the testing purposes.

The GDM template files are part of this repo and should be used for reference for the next alineas.
The following adjustments are made:

 - Adding a service account
 - Include Token
 - Include bearer for each URL which will download the packages from GCS 
 - Change the JSON download path into: 

The **service account** is already included in failover and auto-scale BIG-IP GDM templates. But is not present in stand-alone GDM templates.
```
f5-existing-stack-payg-3nic-bigip.py

'serviceAccounts': [{
            'email': str(context.properties['serviceAccount']),
            'scopes': ['https://www.googleapis.com/auth/cloud-platform']
            }],
```
```
f5-existing-stack-payg-3nic-bigip.py.schema

serviceAccount:
    description: Enter the Google service account to use for autoscale API calls, for example 'username@projectname.iam.serviceaccount.com'.
    type: string
```
```
f5-existing-stack-payg-3nic-bigip.yaml

### Enter the Google service account to use for autoscale API calls, for example 'username@projectname.iam.serviceaccount.com'.
   serviceAccount: <<your service account>>
```

The service account need to get added to the .yaml file as a required value. This makes that all three files need to get adjusted.
The token is to authenticate the service account for making GCS accessible for the service account and only needs to be added to the .py file.
```
f5-existing-stack-payg-3nic-bigip.py

'TOKEN="$(curl "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token" -H "Metadata-Flavor: Google"|cut -d \'"\' -f4)"',
```
Including the bearer and changing the download URI can be done on the same string per package like shown in underneath example.
```
curl -siL --retry 20 -o /config/cloud/f5-cloud-libs.tar.gz -H "Authentication: Bearer ${TOKEN}" https://storage.googleapis.com/storage/v1/b/gert-bucket/o/f5-cloud-libs.tar.gz?alt=media
```
But instead of using above URL structure, the existing curl cmds have been removed and a new piece of code has been written which will check defined ‘gcsURLs’ in .yaml and download these in /config/cloud and start the installation.
```
f5-existing-stack-payg-3nic-bigip.py

'\n'.join(
       [
           'curl -sfL --retry 20 -H "Authorization: Bearer $TOKEN" -o /config/cloud/{} {}'.format(
                  re.findall("([^/]*)\?alt=media$", urllib.parse.unquote(url))[0], url,
                   )
                   for url in context.properties['gcsURLs']
      ]
),
```
```
f5-existing-stack-payg-3nic-bigip.py.schema

gcsURLs:
    description: List of GCS URL files to download of the form 'https://storage.googleapis.com/storage/v1/b/BUCKET_NAME/o/URL_ENCODED_PATH_TO_FILE?alt=media'.
    type: array
    default: []
```
```
f5-existing-stack-payg-3nic-bigip.yaml

### Enter the List of GCS URL files to download of the form 'https://storage.googleapis.com/storage/v1/b/BUCKET_NAME/o/URL_ENCODED_PATH_TO_FILE?alt=media'. Make sure when adding more URLS to respect the array structure ['text','text',text]
   gcsURLs: ['https://storage.googleapis.com/storage/v1/b/gert-bucket/o/f5-cloud-libs.tar.gz?alt=media', 'https://storage.googleapis.com/storage/v1/b/gert-bucket/o/f5-cloud-libs-gce.tar.gz?alt=media', 'https://storage.googleapis.com/storage/v1/b/gert-bucket/o/f5-appsvcs-3.20.0-3.noarch.rpm?alt=media']
```

For this purpose, libraries have been added to the .py file:
•	Import re – which imports the use of regular expressions
•	Import urllib – Fetches URLs

Now everything is ready and configured.

## Testing

Let’s test it by deploying the GDM template in gcloud via:
gcloud deployment-manager deployments create gert-bigip-3nic --config f5-existing-stack-payg-3nic-bigip.yaml --description "gert-bigip-3nic"

After 2 – 3 minutes you will see:

![](png/deployment-completed.png)

Let’s grab the GCP console and go your deployed VM and select the BIG-IP. In the log section check out ‘Serial port 1 (console) read through the log. (if you can’t wait)

After a minute of 5, grab the management public IP address (NIC1) and open SSH to add a password. Your SSH session should include an SSH private key which has been generated.

To change the admin password and save it: **modify auth password** and **save sys config**.

Now use the management public IP address from NIC1 to open a GUI and login to your BIG-IP.

When the BIG-IP GUI opens...

Congratulations, you have succesfully deployed a BIG-IP through a GDM template in an isolated situation.

## Conclusion

When a customer use case requires BIG-IP isolation from the Internet, don’t fear no more. This can be done as the above solution has shown.

Google Cloud Platform provides capabilities to redirect network and DNS to restricted.googleapi.com, making that an isolated VM is able to leverage the Google API and that’s exactly what the customized GDM template needs to start functioning. Deploying a BIG-IP where packages get downloaded from the local Google Cloud Storage and install to make that BIG-IP is up in running in just a few minutes.

Once the BIG-IP is in a running state it can be used to deploy any application service or application security to deliver available and protected apps.

## Troubleshooting

What if things don’t work. Here are some tips to check to understand if things are going wrong and where to look.

1.	Check the mentioned serial port 1 console at the deployed BIG-IP. Your output should look similar to the underneath output which is taken from a successful deployment for your reference.
```
Serial port 1 (console) output for bigip1-gert-bigip-3nic
SeaBIOS (version 1.8.2-20200706_154733-google)
Total RAM Size = 0x0000000780000000 = 30720 MiB
CPUs found: 8     Max CPUs supported: 8
found virtio-scsi at 0:3
virtio-scsi vendor='Google' product='PersistentDisk' rev='1' type=0 removable=0
virtio-scsi blksize=512 sectors=79691776 = 38912 MiB
drive 0x000f2450: PCHS=0/0/0 translation=lba LCHS=1024/255/63 s=79691776
Sending Seabios boot VM event.
Booting from Hard Disk 0...
[    4.732444] piix4_smbus 0000:00:01.3: SMBus base address uninitialized - upgrade BIOS or use force_addr=0xaddr
[    6.291468] common-relabel[1219]: performing intial labeling of common volumes
[[32m  OK  [0m] Started Run F5 sysinit scripts.
[[32m  OK  [0m] Started Serial Getty on ttyS0.
         Starting Serial Getty on ttyS0...
[[32m  OK  [0m] Started F5 Prompt Status Daemon.
         Starting F5 Prompt Status Daemon...
         Starting Swap VE ethernet devices at startup...
         Starting IPv4 firewall with iptables...
[[32m  OK  [0m] Started Getty on tty1.
         Starting Getty on tty1...
[[32m  OK  [0m] Reached target Login Prompts.
         Starting Login Service...
[[32m  OK  [0m] Started Command Scheduler.
         Starting Command Scheduler...
[[32m  OK  [0m] Started boot event collector.
         Starting boot event collector...
[[32m  OK  [0m] Started Login Service.
[[32m  OK  [0m] Started IPv4 firewall with iptables.
         Starting IPv6 firewall with ip6tables...
[[32m  OK  [0m] Started IPv6 firewall with ip6tables.
         Starting Install F5 firewall rules...
[[32m  OK  [0m] Started Install F5 firewall rules.
[[32m  OK  [0m] Started Swap VE ethernet devices at startup.
         Starting Bring up/down networking...
[[32m  OK  [0m] Created slice user-0.slice.


BIG-IP 15.1.0.4 Build 0.0.6
Kernel 3.10.0-862.14.4.el7.ve.x86_64 on an x86_64
bigip1-gert-bigip-3nic login: [[32m  OK  [0m] Started Setup cluster environment..
         Starting Generate ssh host keys (if needed)...
[[32m  OK  [0m] Started Generate ssh host keys (if needed).
         Starting OpenSSH server daemon...
[[32m  OK  [0m] Started OpenSSH server daemon.
         Starting Initialize ssh known_hosts...host and mgmt backplane peers...
[[32m  OK  [0m] Started Initialize ssh known_hosts ...alhost and mgmt backplane peers.
         Starting Mark the point in startup ...nd cluster interfaces are up....
[[32m  OK  [0m] Started Mark the point in startup w... and cluster interfaces are up..
[[32m  OK  [0m] Started Updates availability status based on init level.
         Starting Updates availability status based on init level...
         Starting pkg-tools daemon facilitat...ibility across TMOS instances...
         Starting LSB: syslog-ng-reload hand...arting or reloading syslog-ng...
         Starting start and stop ntpd...
[[32m  OK  [0m] Started pkg-tools daemon facilitate...atibility across TMOS instances.
[[32m  OK  [0m] Started start and stop ntpd.
         Starting start and stop MySQL...
[[32m  OK  [0m] Started start and stop MySQL.
[[32m  OK  [0m] Started LSB: syslog-ng-reload handles starting or reloading syslog-ng.
         Starting The rsync daemon synchronizes files between machines....
[[32m  OK  [0m] Started The rsync daemon synchronizes files between machines..
         Starting dhclient automatically con...rface when DHCP is available....
[[32m  OK  [0m] Started Initialize BIG-IP Virtual Edition.
[[32m  OK  [0m] Reached target vadc-init.
         Starting Initial cloud-init job (pre-networking)...
[[32m  OK  [0m] Started dhclient automatically conf...terface when DHCP is available..
         Starting dhclient automatically con...rface when DHCP is available....
[[32m  OK  [0m] Started dhclient automatically conf...terface when DHCP is available..
         Starting LSB: start and stop Apache HTTP Server...
         Starting Removes kruft files left by pre 11.5 versions of qkview...
[[32m  OK  [0m] Started Removes kruft files left by pre 11.5 versions of qkview.
[[32m  OK  [0m] Started LSB: start and stop Apache HTTP Server.
[[32m  OK  [0m] Started The named process is the DNS server daemon..
         Starting The named process is the DNS server daemon....
[[32m  OK  [0m] Started It is restarting platform d...d ) on mcp/tmm/bigstart restart.
         Starting It is restarting platform ...) on mcp/tmm/bigstart restart...
[[32m  OK  [0m] Started Chassis Management Daemon.
         Starting Chassis Management Daemon...
[[32m  OK  [0m] Started The device management daemo...vice trust group functionality..
         Starting The device management daem...ce trust group functionality....
         Starting RUNIT: init scheme with service supervision...
[[32m  OK  [0m] Started RUNIT: init scheme with service supervision.
[[32m  OK  [0m] Reached target Multi-User System.
[[32m  OK  [0m] Reached target Cloud-init target.
         Starting Update UTMP about System Runlevel Changes...
[[32m  OK  [0m] Started Update UTMP about System Runlevel Changes.
[[32m  OK  [0m] Started Initial cloud-init job (pre-networking).
         Starting Initial cloud-init job (metadata service crawler)...
[[32m  OK  [0m] Started Initial cloud-init job (metadata service crawler).
[[32m  OK  [0m] Reached target Cloud-config availability.
         Starting Apply the settings specified in cloud-config...
[[32m  OK  [0m] Started Apply the settings specified in cloud-config.
         Starting Execute cloud user/final scripts...
[[32m  OK  [0m] Started Execute cloud user/final scripts.
mcpd has reached 'running' state
[ [31m*[1;31m*[0m[31m*  [0m] A stop job is running for LSB: star...pache HTTP Server (6s / 5min 1s)
[K[[31m*[1;31m*[0m[31m*   [0m] A stop job is running for LSB: star...pache HTTP Server (7s / 5min 1s)
[K[[1;31m*[0m[31m*    [0m] A stop job is running for LSB: star...pache HTTP Server (7s / 5min 1s)
[K[[0m[31m*     [0m] A stop job is running for LSB: star...pache HTTP Server (8s / 5min 1s)
[K[[1;31m*[0m[31m*    [0m] A stop job is running for LSB: star...pache HTTP Server (8s / 5min 1s)
[K[[31m*[1;31m*[0m[31m*   [0m] A stop job is running for LSB: star...pache HTTP Server (9s / 5min 1s)
[K[ [31m*[1;31m*[0m[31m*  [0m] A stop job is running for LSB: star...pache HTTP Server (9s / 5min 1s)
[K[  [31m*[1;31m*[0m[31m* [0m] A stop job is running for LSB: star...ache HTTP Server (10s / 5min 1s)
[K[   [31m*[1;31m*[0m[31m*[0m] A stop job is running for LSB: star...ache HTTP Server (10s / 5min 1s)
[K[    [31m*[1;31m*[0m] A stop job is running for LSB: star...ache HTTP Server (11s / 5min 1s)
[K[     [31m*[0m] A stop job is running for LSB: star...ache HTTP Server (11s / 5min 1s)
[K[[32m  OK  [0m] Stopped LSB: start and stop Apache HTTP Server.
         Stopping dhclient automatically con...rface when DHCP is available....
[[32m  OK  [0m] Stopped dhclient automatically conf...terface when DHCP is available..
         Stopping dhclient automatically con...rface when DHCP is available....
[[32m  OK  [0m] Stopped dhclient automatically conf...terface when DHCP is available..
         Stopping The rsync daemon synchronizes files between machines....
[[32m  OK  [0m] Stopped The rsync daemon synchronizes files between machines..
         Stopping start and stop ntpd...
         Stopping pkg-tools daemon facilitat...ibility across TMOS instances...
         Stopping LSB: syslog-ng-reload hand...arting or reloading syslog-ng...
         Stopping Updates availability status based on init level...
[[32m  OK  [0m] Stopped pkg-tools daemon facilitate...atibility across TMOS instances.
[[32m  OK  [0m] Stopped Updates availability status based on init level.
[[32m  OK  [0m] Stopped LSB: syslog-ng-reload handles starting or reloading syslog-ng.
[[32m  OK  [0m] Stopped start and stop ntpd.
[[32m  OK  [0m] Stopped target Remote File Systems.
         Stopping Mark the point in startup ...nd cluster interfaces are up....
[[32m  OK  [0m] Stopped Mark the point in startup w... and cluster interfaces are up..
         Stopping Initialize ssh known_hosts...host and mgmt backplane peers...
[[32m  OK  [0m] Stopped Initialize ssh known_hosts ...alhost and mgmt backplane peers.
         Stopping OpenSSH server daemon...
[[32m  OK  [0m] Stopped OpenSSH server daemon.
         Stopping Generate ssh host keys (if needed)...
[[32m  OK  [0m] Stopped Generate ssh host keys (if needed).
         Stopping Setup cluster environment....
[[32m  OK  [0m] Stopped Setup cluster environment..
         Stopping XenServer Virtual Machine ...ing host integration services...
         Stopping Initializes FIPS devices...
[[32m  OK  [0m] Stopped XenServer Virtual Machine d...iding host integration services.
[[32m  OK  [0m] Stopped Initializes FIPS devices.
[[32m  OK  [0m] Stopped target Network is Online.
         Stopping Bring up/down networking...
[[32m  OK  [0m] Stopped Bring up/down networking.
         Stopping Swap VE ethernet devices at startup...
         Stopping IPv6 firewall with ip6tables...
[[32m  OK  [0m] Stopped Swap VE ethernet devices at startup.
[[32m  OK  [0m] Stopped IPv6 firewall with ip6tables.
         Stopping IPv4 firewall with iptables...
[[32m  OK  [0m] Stopped IPv4 firewall with iptables.
         Stopping Run F5 sysinit scripts...
[[32m  OK  [0m] Stopped Run F5 sysinit scripts.
[[32m  OK  [0m] Stopped Run F5 module sysinit scripts.
         Stopping Run F5 module sysinit scripts...
[[32m  OK  [0m] Stopped Dump dmesg to /var/log/dmesg.
         Stopping Dump dmesg to /var/log/dmesg...
         Stopping Transfer entropy from various sources to kernel....
[[32m  OK  [0m] Stopped Transfer entropy from various sources to kernel..
[[32m  OK  [0m] Stopped target Basic System.
[[32m  OK  [0m] Stopped target Paths.
[[32m  OK  [0m] Stopped target Slices.
[[32m  OK  [0m] Removed slice User and Session Slice.
[[32m  OK  [0m] Stopped target Sockets.
[[32m  OK  [0m] Closed D-Bus System Message Bus Socket.
[[32m  OK  [0m] Stopped target System Initialization.
         Stopping Update UTMP about System Boot/Shutdown...
[[32m  OK  [0m] Stopped target Encrypted Volumes.
[[32m  OK  [0m] Stopped Update is Completed.
         Stopping Update is Completed...
         Stopping Load/Save Random Seed...
[[32m  OK  [0m] Stopped Load legacy module configuration.
         Stopping Load legacy module configuration...
[[32m  OK  [0m] Stopped Rebuild Hardware Database.
         Stopping Rebuild Hardware Database...
[[32m  OK  [0m] Stopped Setup Virtual Console.
         Stopping Setup Virtual Console...
[[32m  OK  [0m] Stopped Apply Kernel Variables.
         Stopping Apply Kernel Variables...
[[32m  OK  [0m] Stopped Load Kernel Modules.
         Stopping Load Kernel Modules...
[[32m  OK  [0m] Stopped target Swap.
         Deactivating swap /dev/disk/by-uuid...b-2495-4213-99ff-01199ea3badf...
[[32m  OK  [0m] Stopped Initialize common volume security labels.
         Stopping Initialize common volume security labels...
[[32m  OK  [0m] Stopped Rebuild Journal Catalog.
         Stopping Rebuild Journal Catalog...
[[32m  OK  [0m] Deactivated swap /dev/vg-db-vda/dat.swapvol.
[[32m  OK  [0m] Deactivated swap /dev/mapper/vg--db--vda-dat.swapvol.
[[32m  OK  [0m] Deactivated swap /dev/disk/by-id/dm...uFmcdgn3RpzeYuhpEJ72FIAZ2bUHkHu.
[[32m  OK  [0m] Deactivated swap /dev/disk/by-id/dm-name-vg--db--vda-dat.swapvol.
[[32m  OK  [0m] Deactivated swap /dev/dm-4.
[[32m  OK  [0m] Stopped Update UTMP about System Boot/Shutdown.
[[32m  OK  [0m] Stopped Load/Save Random Seed.
[[32m  OK  [0m] Deactivated swap /dev/disk/by-uuid/...cab-2495-4213-99ff-01199ea3badf.
         Stopping Security Auditing Service...
[[32m  OK  [0m] Stopped Security Auditing Service.
[[32m  OK  [0m] Stopped Create Volatile Files and Directories.
         Stopping Create Volatile Files and Directories...
[[32m  OK  [0m] Stopped Flush Journal to Persistent Storage.
         Stopping Flush Journal to Persistent Storage...
         Unmounting Mount TMOS 'log' filesystem...
[[32m  OK  [0m] Stopped target Local File Systems.
         Unmounting /config...
         Unmounting /appdata...
         Unmounting /dev/mprov/tmm...
         Unmounting /var/tmstat...
         Unmounting /var/prompt...
[[32m  OK  [0m] Stopped Configure read-only root support.
         Stopping Configure read-only root support...
         Unmounting /shared/rrd.1.2...
         Unmounting /run/user/91...
         Unmounting /var/loipc...
[[32m  OK  [0m] Failed unmounting Mount TMOS 'log' filesystem.
[[32m  OK  [0m] Unmounted /appdata.
[[32m  OK  [0m] Unmounted /dev/mprov/tmm.
[[32m  OK  [0m] Unmounted /var/tmstat.
[[32m  OK  [0m] Unmounted /var/prompt.
[[32m  OK  [0m] Unmounted /config.
[[32m  OK  [0m] Unmounted /shared/rrd.1.2.
[[32m  OK  [0m] Unmounted /run/user/91.
[[32m  OK  [0m] Unmounted /var/loipc.
         Unmounting Mount TMOS 'share' filesystem...
         Unmounting /var...
[[32m  OK  [0m] Failed unmounting /var.
[[32m  OK  [0m] Unmounted Mount TMOS 'share' filesystem.
[[32m  OK  [0m] Reached target Unmount All Filesystems.
[[32m  OK  [0m] Stopped target Local File Systems (Pre).
[[32m  OK  [0m] Stopped Create Static Device Nodes in /dev.
         Stopping Create Static Device Nodes in /dev...
[[32m  OK  [0m] Stopped Remount Root and Kernel File Systems.
         Stopping Remount Root and Kernel File Systems...
         Stopping LVM2 PV scan on device 8:3...
         Stopping Monitoring of LVM2 mirrors... dmeventd or progress polling...
[[32m  OK  [0m] Stopped Monitoring of LVM2 mirrors,...ng dmeventd or progress polling.
[[32m  OK  [0m] Stopped LVM2 PV scan on device 8:3.
[[32m  OK  [0m] Removed slice system-lvm2\x2dpvscan.slice.
[[32m  OK  [0m] Reached target Shutdown.
dracut Warning: Killing all remaining processes
Rebooting.
[  144.306133] Restarting system.
SeaBIOS (version 1.8.2-20200706_154733-google)
Total RAM Size = 0x0000000780000000 = 30720 MiB
CPUs found: 8     Max CPUs supported: 8
found virtio-scsi at 0:3
virtio-scsi vendor='Google' product='PersistentDisk' rev='1' type=0 removable=0
virtio-scsi blksize=512 sectors=79691776 = 38912 MiB
drive 0x000f2450: PCHS=0/0/0 translation=lba LCHS=1024/255/63 s=79691776
Sending Seabios boot VM event.
Booting from Hard Disk 0...
[    4.667099] piix4_smbus 0000:00:01.3: SMBus base address uninitialized - upgrade BIOS or use force_addr=0xaddr
[[32m  OK  [0m] Started Run F5 sysinit scripts.
         Starting IPv4 firewall with iptables...
[[32m  OK  [0m] Started F5 Prompt Status Daemon.
         Starting F5 Prompt Status Daemon...
         Starting Login Service...
         Starting Swap VE ethernet devices at startup...
[[32m  OK  [0m] Started Getty on tty1.
         Starting Getty on tty1...
[[32m  OK  [0m] Started Command Scheduler.
         Starting Command Scheduler...
[[32m  OK  [0m] Started boot event collector.
         Starting boot event collector...
[[32m  OK  [0m] Started Serial Getty on ttyS0.
         Starting Serial Getty on ttyS0...
[[32m  OK  [0m] Reached target Login Prompts.
[[32m  OK  [0m] Started Login Service.
[[32m  OK  [0m] Started IPv4 firewall with iptables.
         Starting IPv6 firewall with ip6tables...
[[32m  OK  [0m] Started IPv6 firewall with ip6tables.
         Starting Install F5 firewall rules...
[[32m  OK  [0m] Started Install F5 firewall rules.
[[32m  OK  [0m] Started Swap VE ethernet devices at startup.
         Starting Bring up/down networking...
[[32m  OK  [0m] Created slice user-0.slice.


BIG-IP 15.1.0.4 Build 0.0.6
Kernel 3.10.0-862.14.4.el7.ve.x86_64 on an x86_64
bigip1-gert-bigip-3nic login: mcpd has reached 'running' state
```

2.	After 10 minutes, check if you can see the login screen of the BIG-IP by using NIC0, instead of NIC1. This implies that the NIC swap hasn’t took place. When you want to know this sooner use SSH on NIC1. You might need to add tcp/22 to your external fw rule on GCP.
3.	To understand what goes wrong you can login to the BIG-IP on NIC0 via SSH and check if the cloud-lib and AS3 packages are downloaded in /config/cloud
4.	Check at tail -f /var/log/cloud/google/install.log and check if the cloud-init is still busy. When this is still the case after 5 minutes of deploying, this is a bad sign.
5.	To understand if the token and bearer are functioning and the BIG-IP is able to download the files stored in GCS. From the BIG-IP SSH request a token: 
```
curl -sf -H 'Metadata-Flavor: Google' 'http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token' | jq -r .access_token
```
![](png/get-token.png) 

use the token to download a file form GCS:
```
curl -fL --retry 20 -o /config/cloud/gce/f5-appsvcs-3.20.0-3.noarch.rpm -H 'Authorization: Bearer copy_your_access_token_here' 'https://storage.googleapis.com/storage/v1/b/gert-bucket/o/f5-appsvcs-3.20.0-3.noarch.rpm?alt=media'
```
 
As a result, you should see the file getting downloaded. 

![](png/download-package.png)

When this action returns a 403, your authentication is not correct and you should check the authorization of your service account for your GCS bucket.

## References
 - [Configuring Private Google Access](https://cloud.google.com/vpc/docs/configure-private-google-access)
 - [Setting up private connectivity to Google APIs and services](https://cloud.google.com/vpc-service-controls/docs/set-up-private-connectivity)
 - [BIG-IP 3NIC PAYG Existing GDM template](https://github.com/F5Networks/f5-google-gdm-templates/tree/master/supported/standalone/3nic/existing-stack/payg)
 - [f5-google-bigip-isolated-vpcs](https://github.com/memes/f5-google-bigip-isolated-vpcs)


