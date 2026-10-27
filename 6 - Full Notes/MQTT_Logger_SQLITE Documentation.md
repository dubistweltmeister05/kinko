[[RhyGen]]

![[mqtt_logger_sqlite_documentation]]
## decode_payload
1. First, we save the incoming data via MQTT in a tuple called `unpacked`.

2. Then, we start slicing through it all. The first member of the log structure is the elec_db. This has the active and reactive power data - all being `floats` in nature. These can be found at indexes 1 through 9 of the unpacked tuple. Index 0 has the modbus timestamp, and raw_apps is found at index 18. The rest, needs to be scaled. 

3. We then call the decode_record function from the decode_logs script, to obtain the decoded and scaled values of the mech parameters that are to be logged. 

4. Post that, we run another loop, this time to round off the elec params to 1 decimal place, and save it all to the tuple that we shall return called record. 

## archive_to_parquet
1. Open the database file that we are trying to log to, via the db_path argument that has been passed to this function.

2. Get all logs from the database file into a pandas dataframe. Validate via a null check.

3. Creates a new parquet file with the filename as `YYYYMMDD_HHMMSSffffff` . parquet

4. Write to it via the `df.to_parquet()` function.

5. Close the connection once write is complete
## main
1. We have a bunch of argument options that are available to us while we call this script. the defaults have been set to what we have flashed to the MCU. 
   
2. A Python subprocess is opened next, where we run a command that connects to the MQTT pipeline that we have set up, and incoming MQTT data is piped into the subprocess that is triggered. All operations that follow are done on whatever is returned by the stdout (whatever is piped to the subprocess that we ran up above).  

3. We strip all that was received, with a whitespace being the separator that we use. Upon collecting the payload, we convert it from hex to bytes. The payload shall hav eno whitespaces in between it, so we are good on that. 

4. Then, unpack that incoming payload, and use a list of  variables to store whatever is being unpacked from the payload.  Use a check for the headers and see if the formatting matches what we expect to be coming in. If any of the fields are found to be wrong, don't bother processing anything else and just move on to the next iteration of the loop. 

5.  Find out what records are coming in for a particular device ID. If it is an old record sequence, skip processing anything. If it is somewhere in between, process only the new ones and skip the old. If all new, process everything. 

6. Compute the receive timestamp for the incoming batch, and use it to generate the folder structure required by the data reader. Create the run metadata JSON file if it doesn't already exist, then open the SQLite database corresponding to the device, date, and run ID.

7. Create a list to store the decoded records before writing them to the database. Iterate through the records in the batch, starting from the `skip` index, calculate the byte offset for each record, extract the corresponding bytes from the payload, and decode them using `decode_payload()`. Append each decoded record to the list.

8. If there are records to insert, convert the list of dictionaries into a Pandas DataFrame and append its contents to the `logs` table in SQLite. Then, query the database to find the total number of rows currently stored, close the database connection, and print a message showing how many records were saved and the current total.

9. Once the database reaches the configured maximum row count, archive its contents to Parquet using `archive_db_to_parquet()`. This allows the logger to move accumulated records into a format more suitable for data analysis and plotting.

10. If the user interrupts the script with `Ctrl+C`, catch the `KeyboardInterrupt` and print a shutdown message. The `finally` block terminates the `mosquitto_sub` subprocess so that it doesn't continue running after the Python script stops.