# WhatsOp Change Management

Credit: Gabi Zapodeanu (see https://github.com/zapodeanu/DevNet_Create_2018)

**NetDevOps Engineer Everyday Skills**

This repo will include all the files needed for the DevNet Create 2018 Workshop.

Do you want to learn how to write simple ChatOps apps for IOS XE network devices? This session will explore a few IOS XE device programmability capabilities to help you create your first WhatsOp application using Guest Shell.

**This workshop requires:**

 - Cisco DevNet account
 - Webex Teams account – sign up at https://teams.webex.com/
 - GitHub account
 - DevNet CSR1000V sandbox – provided, or reserved by you here: https://developer.cisco.com/site/sandbox/
 - You will need Python 3.x installed
 - Webex Teams SDK

**The repo includes these files**

 - config.py - configuration file that includes account usernames and passwords
 - eem_cli_config.txt - cli configuration for the CSR1000V router that will be used during the workshop
 - save_base_config.py - script to establish the baseline configuration
 - config_change.py - application code
 - the workshop PowerPoint Presentation

 **Application Workflow**

 - User makes IOS XE device configuration change
 - Syslog triggers EEM Guest Shell Python script execution
 - The config_change.py script will:
   - Detect if a configuration change and what changed
   - Collect the device hostname
   - Identify the user the made the change using Python CLI
   - Create Webex Teams room using REST APIs, invite Approver to room, post the above information to ask for approval
   - If changes approved, save new configuration as baseline
   - If not approved or no response, rollback to the previous baseline configuration
   - Close the Webex Teams room in 30 seconds

## Instructions
1) Enable IOX service:
```
Cat9300#conf t
Cat9300(config)#iox  
```
2) Wait for the service to fully come up (all services must be in 'Running' state):
```
Cat9300#show iox-service

IOx Infrastructure Summary:
---------------------------
IOx service (CAF)    : Running
IOx service (HA)     : Running
IOx service (IOxman) : Running
Libvirtd             : Running
```
3) Configure the guest-shell's interface:
```
conf t
 app-hosting appid guestshell
   app-vnic management guest-interface 0
```
4) Start guest-shell service by running CLI:
```
Cat9300#guestshell enable
Management Interface will be selected if configured
Please wait for completion
Guestshell enabled successfully
```
5) Confirm that guestshell is up and running:
```
Cat9300#show app-hosting list
App id                           State
------------------------------------------------------
guestshell                       RUNNING
```
6) If proxy is required:
You can create proxy_vars.sh file locally and copy it to switch flash, or use guestshell and VI editor to create file directly on the switch.
```
export http_proxy=http://10.100.100.1:8080
export https_proxy=http://10.100.100.1:8080
export HTTP_PROXY=http://10.100.100.1:8080
export HTTPS_PROXY=http://10.100.100.1:8080
```
also assure that these variables are loaded each time guestshell is executed by adding to file .bashrc (sudo vi .bashrc) the following line:
```
source /bootflash/proxy_vars.sh
```
7) Configure DNS server (use VI editor to create file and add content):
```
[guestshell@guestshell ~]$ sudo vi /etc/resolv.conf
!
! to edit file, press 'i', copy-paste line below
nameserver 172.18.108.43
!
! to save press Esc and then combination :wq!
"/etc/resolv.conf" 1L, 25C written
```
8) Install Python's Webex Teams SDK:
```
[guestshell@guestshell ~]$ sudo pip install webexteamssdk --proxy <PROXY IF RELEVANT>
Collecting requests
...
```
9) Install Git on Guestshell:
```
[guestshell@guestshell ~]$ sudo yum install git
```
10) Clone the repo to the switch/router:
```
[guestshell@guestshell ~]$ cd /bootflash/guest-share/
[guestshell@guestshell guest-share]$ git clone https://github.com/obrigg/WhatsOp.git
Cloning into 'WhatsOp'...
remote: Enumerating objects: 9, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 80 (delta 2), reused 0 (delta 0), pack-reused 71
Unpacking objects: 100% (80/80), done.
```
11) Prepare to Execute 1st Time the App (creating the initial base-config):
```
Cat9300#write mem
Cat9300#guestshell run bash
[guestshell@guestshell ~]$ cd /bootflash/guest-share
[guestshell@guestshell guest-share]$ mkdir CONFIG_FILES
[guestshell@guestshell guest-share]$ ls
CONFIG_FILES                
WhatsOp
…

[guestshell@guestshell guest-share]$ cd WhatsOp
[guestshell@guestshell WhatsOp]$ python save_base_config.py
```
12) Edit the config.py file with your details:
```
sudo vi config.py
```
13) Create the triggering EEM script:
```
conf t

!
event manager applet config_change
event syslog pattern "SYS-5-CONFIG_I" maxrun 240
action 0 cli command "enable"
action 1 cli command "guestshell run python /bootflash/guest-share/WhatsOp/config_change.py"
action 2 cli command "exit"
action 3 cli command "exit"
!
End
```
