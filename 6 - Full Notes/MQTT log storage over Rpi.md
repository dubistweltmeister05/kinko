[[RhyGen]]

The issue with parquet files is that you cannot append to them at the end just like you can with a csv or a txt. It is so by design, since it is meant to be an immutable columnar file, and is designed to be Write Once, Read Many (WORM) kind of a file for big datasets and their analysis. 

Now, to add something to it, we have to open it, load all contents to the RAM, navigate to the end, and then add our data.

Doing this for a small file, on a laptop is still fine. But running that on an RPi, on a file that has hundreds of thousands of logs, can lead to problems - a risk I am not willing to take.

Implemented strategy -
Save active logs to an SQLite db called active_log.db . The advantage of doing this over saving to a csv is that it autosaves itself on every write. So even if the Rpi loses power, the active_log db still retains all that was written to it previously.

once a day, at a fixed time, we shall convert the active_log .db file to a parquet, delete the existing active_log.db, and make a new one, restarting the logging process. We get 1 parquet file per day, that is timestamped within the naming format as - `mqttlog_<date>_<last_logged_timestamp_IST>.parquet`