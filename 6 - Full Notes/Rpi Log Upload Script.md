[[RhyGen]]

## Prompt
This is a data reader script that runs on a r-pi, at port 8501, via a system service file called data-reader.service. The script resides at ~/Desktop/bibtya_logs/script on the Rpi, and the data to be read from resides at knv@raspberrypi:~/Desktop/bibtya_logs/data $ ls 
'mode=dyno'  'mode=engine'  'mode=fault'  'mode=road' 
Every time I need to upload new collected logs to the rpi, i need to manually scp the whole log folder structure. I need a way to simplify this. I am thinking that we can have a web page that anyone on the same network as the RPi can open up, say on port 8502.

The web page script shall take the user's local directory as an input, and shall compare with the data directory on the Rpi. It shall then copy whatever is new on the user's machine, WHILE PRESERVING THE ALREADY EXISTING LOGS. The preservation of old logs is important, because the user's machine may not have some old log runs, but that doesn't mean we should just delete the old logs from the RPi. 

After it is done copying, it shall restart the system service file data-reader.service that is setup to run on the RPi, since we have put on new data and the data reader needs a refresh for it to show up and be plotted.

Once we have the script, we can make it a system service as well, so that it shall execute autonomously and on boot, without me having to ssh into the Rpi and manually turn it on. 