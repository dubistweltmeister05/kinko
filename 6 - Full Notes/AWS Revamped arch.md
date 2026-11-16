[[RhyGen]]
## Architecture
We are running the server components as Docker Compose Services. What is docker compose? A YAML file that let's us define the params for multi-container setups. *Question - why are we running these services / components in multiple components?* 

*Answer - We can do this for a single container while we define all the needs for each service that is needed to be run in that container. This works like a requirements. txt file essentially, where we describe what's needed, and it shall be done.* 


Ingester - it is mentioned that we are uploading a raw data and a parquet chunk per device to S3. The whole reason that we are using parquet is that it is easy to read from when it comes to big data. And the structure of the file is such that it becomes tougher to write to it as the file keeps getting bigger. So, if we are having a parquet file (assuming that's what is called as a chunk here), once every 10 minutes, that'll lead to a bunch of files per device, which also kinda defeats the purpose of having parquet as the choice of log file to begin with. 

My suggestion shall be that we save raw chunks once every 10 minutes, and run a job that converts the last <insert_time_interval> (eg 6 hours) of raw data into parquet files. The nightly jobs server the same purpose as I thought here, so that's alright. 

## Load and Sizing
### Data Per Device
The data that is being pushed per device is as follows - 
```
| Period  | Publishes | Log data  | Headers |   Total payload         |
| ------- | --------: | --------: | ------: | ----------------------: |
| Minute  |        30 |  38,400 B |   480 B |   38.9 KB               |
| Hour    |     1,800 |   2.30 MB | 28.8 KB |   2.33 MB               |
| Day     |    43,200 |   55.3 MB | 0.69 MB |   56.0 MB   (53.4 MiB)  |
| 30 days | 1,296,000 |   1.66 GB | 20.7 MB |   1.68 GB               |
```

### Passwords and Usernames
The username shall be the deviceID that we have set on the device via the UID. The password is a random 32-char string that shall be unique to the device. 

Broker shall keep hashed passwords and plain copies shall be in the SSM Parameter store as encrypted Secure Strings, at `/rhygen/devices/<ID>/mqtt_password`. 



## 12Questions for you

Answers needed to keep the dates. Items marked blocking hold up the dates in section 11 until answered.

1.  **Domain.** A new domain or a subdomain of one you already own? Is DNS in Route 53 okay? Who holds the registrar account?
   We can go for a new domain. I shall make a registrar account via cloudflare's service. 
   
2.  **AWS access.** Access to the account for our team (Identity Center preferred), and who receives alert and billing emails.
   We shall make an IAM user for you with permissions granted as needed for the setup. The alert and billing emails shall go to `sajal.deolikar@rhygentech.com` and `kshitij.vaze@rhygentech.com` .
   
3.  **Firmware vNext.** Can the firmware with the domain, TLS, password, CONNACK check and the 48-byte header be on a bench logger by Fri 20 Nov, when the infrastructure is ready? Who builds and flashes it?   
   As for the device credentials, we can have a python script that uses the device ID to generate a password, and this shall be added to an allowed list username-password key-value pairs on the server. We can include the device username and password in the command to connect to the server that is sent from the device. Once the server acknowledges this, we can start sending the logged data. I shall be testing and flashing it to the bench logger. As for timeline, these changes shall go beyond 20th November. At the moment, 23rd November shall be the tentative deadline for completing the vNext for the firmware.  
   
4.  **EC200 TLS.** Which EC200 firmware build is on the loggers? Does it support loading a CA file, SNI, and hostname checking? How does it handle certificate dates before its clock is set? 
   As mentioned above, we are planning to not have TLS on the devices at the moment, so we can skip this part.

5.  **Decoder inputs.** The record field layout with the J1939 and Modbus scaling table, the three current scripts, and a few raw message captures from both devices.
   I have attached the python scripts that decode the incoming data, and the dashboard scripts that are in use at the moment. I will share a couple logs with you shortly. 
   
6. **Record size.** Please confirm a record is 64 _bytes_ (1,296 bytes per 20-record publish). It was mentioned as 64 KB in discussion, and every storage and cost figure here depends on it.
   The data that is being pushed per device is as follows - 
```
| Period  | Publishes | Log data  | Headers |   Total payload         |
| ------- | --------: | --------: | ------: | ----------------------: |
| Minute  |        30 |  38,400 B |   480 B |   38.9 KB               |
| Hour    |     1,800 |   2.30 MB | 28.8 KB |   2.33 MB               |
| Day     |    43,200 |   55.3 MB | 0.69 MB |   56.0 MB   (53.4 MiB)  |
| 30 days | 1,296,000 |   1.66 GB | 20.7 MB |   1.68 GB               |
```

7. **Runtime.** Do loggers run 24/7, or only while the engine runs? This changes storage estimates and when a device counts as "silent".
   Yes, the loggers are expected to run 24/7 - as long as they have an active source of power. 
   
8. **ML outputs.** Should the filtered time series also be saved to S3 (`exports/`), or only sent to the simulator?
   We can plan this later, as we are yet to work on the ML pipeline for these devices.
   
9. **Dashboard.** Who uses it, and which channels and views matter most for the first pages?
   The dashboard shall be used by anyone with a link to it or access to the EC2 instance that it is running on, and technically, all logged fields matter for the view. We can have a scrolling dashboard that lets the viewer see all the fields.
   
10. **Retention and residency.** Any duty to delete data after a period? Must data stay in India? (This plan keeps both copies in India.)
    The data is intended to stay in India, and we can delete data after a year. 
11. **Alerts.** Who receives device-silent notices, and are business-hours responses acceptable?
    The dead device alert shall be sent to `sajal.deolikar@rhygentech.com` and `kshitij.vaze@rhygentech.com` , and the responses can come as they occur and not just in business-hours.
12. **SD card history.** Do you want gaps in old data filled from the SD cards during the reflash?
	
13. **Dashboard access.** Which GitHub organization should hold the four repositories (section 9), and who should have the dashboard login?
    The organization `https://github.com/Rhygen-Tech` shall hold the repos and any member of this org should have access to the dashboards. 
14. **Budget.** Is about $29–37/mo now and $34–42/mo at 30 devices within range? We'll set the budget alarm to match.
    No issues with the budget. 
15. **SIM plans.** About 2 GB per device per month. Do current plans cover that?
	    Yes, they do.