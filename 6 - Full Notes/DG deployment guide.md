[[RhyGen]]
# Connections
## Power
The MFM and the Logger work on 12V DC power supply. Ideally, an SMPS is needed if there's only mains output available at the site. If not, a buck converter shall be used to 
## J1939
This requires a simple CANH and CANL connection, from the DG's CAN bus to our logger. We also have to be mindful that the DG's CAN bus publishes at 250kbps, and out firmware's default CAN bitrate is set to 500KBPS. Hence, any production releases for the logger, needs to have it's CAN bitrate matched at 250KBPS.     
## Rs485
The MFM has an RS485 connector, that is demarcated with a + and - label on it's connection terminal, while our logger is demarcated with an A+ and B- at the  screw terminal. We have to match the MFM's wiring with our logger's terminals, and we should be good to go. 

# Placement
## MFM placement

## CT Placement
## Logger Placement


