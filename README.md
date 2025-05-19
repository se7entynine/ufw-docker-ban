### 1.) ASN LIST Block List
Notice. as of 4/15/2025 I have been introduced to "aggregate6" https://pypi.org/project/aggregate6/ which can take a list of IP addresses and combine multiple addresses into a larger subnet to reduce the number of lines. I just ran the ASN update, and before aggregation, the script downloaded 84,712 subnets. After aggregation, the EXACT SAME BLOCK LIST now only contains 27,765 subnets, that is a sivings of 56,947 lines, or 67.2% reduction in size!

for example, the OLD ASN list contained the following lines:
```
1.12.0.0/14
1.12.0.0/18
1.12.0.0/20
1.12.14.0/23
1.12.14.0/23
1.12.34.0/23
1.12.64.0/18
1.12.128.0/18
1.12.192.0/18
1.13.0.0/18
1.13.64.0/18
1.13.128.0/18
1.13.192.0/18
1.14.0.0/18
1.14.64.0/18
1.14.128.0/18
1.14.192.0/18
1.15.0.0/18
1.15.64.0/18
1.15.128.0/18
1.15.192.0/18
```

all of these lines are exactly the same as the single line in the new file ```1.12.0.0/14``` as the /14 subnet has a mask of ```0.3.255.255```. This means this single line is good from address ```1.12.0.0``` through address ```1.15.255.255```. The last line above ```1.15.192.0/18``` with a subnet of /18 has a mask of ```0.0.63.255``` which means it goes all the way to ```1.15.255.255``` which is why it is included in the aggregation. 



I block the ASN address ranges of a large number of server rental companies as a lot of "bad actors" use these servers to perform port scans and brute force attacks. 

```ASN_LIST.txt``` --> list of the ASNs I block on my Fortigate SSL VPN loop back interface. This shows the names of the ASN and the revision history tracking of when i added new ASN entires 

```ASN_Update.sh``` combined with ```ASN.txt``` --> script I use to pull all of the IP address details for all ASNs in ```ASN.txt``` and save the results into ```asn_blockX.Y.txt``` files so I can use my fortigate's external threat feeds to import the results. The script downloads (as of 4/15/2025) 27,765 subnet ranges, some of the ranges go as large as a /10 subnet! The ```ASN.txt``` is the raw listing of the blocked ASNs used by the shell script, while ```ASN_List.txt``` is the user-readable and revision history details of the ASNs being blocked as previously detailed. 

```asn_blockX.Y.txt``` --> these are the resulting files made when running the ```ASN_Update.sh``` script. any one Fortigate external threat feed can only handle 131,000 entries, and the script ensures the files are maxed out and aggregates everything into as few files as possible

```manual_block_list.txt``` --> list of IPs that have tried to force a username/password on my fortigate but their ASN is a large telecomm provider and I do not wish to block the entire ASN. this is used on my Fortigate SSL_VPN loop back interface

### 2.) Web Filter Blocks
While the fortigate firewalls do have built in web-filters for advertisements and known malicious actors, it is not blocking everything I would like it to. As such I wanted to use the plethora of Pie-Hole block lists, especially the lists at this amazing site https://firebog.net/. The issue is that these lists are not formatted in the way the Fortigate external threat feeds will accept. As a result I made a script that will download all of the separate lists, format the entries to be compatible with the external threat feeds, and save the entries into separate files with 131,000 entries per file since that is the limit of the threat feeds. 

```webblock.sh``` --> This script pulls the domain names used in multiple Pie-Hole DNS block lists contained in ```web_block_source.txt```. The script formats the data in a way compatible with the fortigate since pie hole lists are formatted as HOST files. The script also performs a little more filtering, but most importantly to removes duplicate entries. For example, currently the PHP script downloads (as of 4/2/2025) 2,519,190 entries and after removing duplicates, has 1,766,773   unique entries being blocked. 

I then use the WEB filter profile within my Fortigate firewall with the resulting ```web_blockX.txt``` files as external threat feed to block significant amounts of ads, tracking, and malicious sites on top of what fortinet already blocks. Refer to ```SSL_VPN Config with loopback and auto-block.txt``` for how I configured my Fortigate SSLVPN. 

```web_blockX.txt``` --> these are the resulting files made when running the ```webblock.sh``` script. any one Fortigate external threat feed can only handle 131,000 entries, and the script ensures the files are maxed out and aggregates everything into as few files as possible. As of 2/4/2025, there are 14x files starting from ```web_block0.txt``` through ```web_block13.txt``` as the 15th file is now empty. 

### 3.) Linux server UFW firewall ASN blocking and Geography blocking

To increase the security of the VPS I am using i also want to be able to use my ASN lists so I can add IP addresses to the ufw firewall to block connections, but I needed a easy way to add and or remove entries from the firewall as required. 

```ufw_update.sh``` combined with ```ASN.txt``` and ```geoblock.txt``` was written to allow for this. 

If ASN based blocking is not desired, it can be turned off by setting the script variable ```block_ASN=1``` from a value of 1 to a value of 0. 

The script will also download many different country code IPv4 IP ranges to allow for Geo based blocking. I have disabled all IPv6 incoming connections to my VPS, so I am not downloading the IPv6 addresses ranges. Currently the script downloads ALL countries IPv4 ranges EXCEPT for the United states and Germany. The US because that is the country I live in, but did not want to block Germany as Hetzner is a German company and I did not want to risk issues. 

The script gets the needed Geo IP ranges from this repository: https://github.com/herrbischoff/country-ip-blocks/tree/master

If Geo based blocking is not desired, it can be turned off by setting the script variable ```block_geo=1``` from a value of 1 to a value of 0. 

If IPv6 incoming connections are not being allowed at all, like I am with my VPS, then keep the variable ```ipv6=0``` set to 0. If you wish to use IPv6, then set this variable to 1. 

to see what the script will attempt to do to your system's UFW configuration without actually applying those changes, set the variable ```test_mode=0``` to a value of 1. The script will output what it is doing, but not actually make changes to UFW. 

when this script is run, it will download all of the ASN and Geo block files, aggregate them into one file, and check each address to see if it currently being blocked by UFW. If it NOT currently being blocked, then it will be added to your UFW config as a "DENY IN" to "ANYWHERE"

the second thing the script will do is compare all of the DENY IN entries in the UFW configuration and confirm they are still part of the ASN/GEO block lists. If the entry is NOT in those lists, then it will be removed. This is to ensure as those lists update over time, both new addresses will be added, but old addresses will be removed as well from your configuration. 

Any "ALLOW IN" entries on the UFW fire wall will be ignored by this script. 

running this script for the first time will take 2-3 days to complete the first time it is run as well over 50,000 entries will be added! The process starts out somewhat fast, but as more entries are added, it takes the UFW subsystem longer and longer to add new additional entries slowing the process down. Just let the script run. 

After the first time it is run, It is suggested to add a line to crontab to run the script once per month, this should only take a few minutes to an hour or two depending on how many addresses need to be added or removed from your UFW configuration. 

The configuration uses a local in policy to allow ONLY addresses in the United States. 
