[[Rhygen Server]]

This directory holds all the config data for the Mosquitto Broker that we have set up for the loggers. 
## ACL
The Access Control List. This is used to control which client can read, write, or be denied access to a particular topic. For the moment, we shall have the devices set-up to only write to certain topics that shall be configured. 

The file itself is pretty human readable, there is nothing much to document here. It states the keyword topic, access control method (read/write/readwrite), and the actual topic that we are configuring. 

## Bridge Config Template
This is used to get the data from the old server to the new one. The new server connects to the old one as a client, and gets the server to get data onto itself. The bridge pulls data **in only**, for the `/log` topics only, at QoS 1. It sends nothing back to the old broker and leaves `/status` and `/cmd` alone.

## Mosquitto . Conf
This is the broker config file. We use this to connect to the Mosquitto broker on the server, configure it with the appropriate params like the port, the ACL file path, the password file for logging into the connection, and other config parameters. 

## DockerFile
- It starts from the official Mosquitto image with an exact version pinned. That image's default command runs `mosquitto -c /mosquitto/config/mosquitto.conf`, so the copied config takes effect automatically and no `CMD` is needed.
- It adds two files, both owned by the `mosquitto` user:
    - **`mosquitto.conf`**: a single listener on port 1883, no anonymous clients, `use_username_as_clientid`, persistence on, the queue and size limits, and logging to stdout. It also points at `/mosquitto/config/passwd` and `/mosquitto/config/acl`, and loads `include_dir /mosquitto/config/conf.d`.
    - **`acl`**: the per-device topic rules, with mode 0600 because newer Mosquitto versions refuse ACL files that others can read (hard rule 8).