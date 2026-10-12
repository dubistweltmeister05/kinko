[[RhyGen]]

# Logging Mechanical params - J1939 over CAN
The J1939 Protocol is nothing but a message packet specification that is transmitted over CAN. We have an elaborate routing system that runs in the app that handles the filtering and specifically listens to the CAN IDs that we have specified in the app's init struct. 
 
We are accepting all messages that are on the bus, without filtering anything, into the CAN peripheral. We are then filtering for the PGNs that we are interested in, via our own application software. This is done because 

# Logging Electrical params - MODBUS RTU via RS485 over UART
## Modbus interaction with Tor's MFM
For this, we are sending a modbus request - handled by  `app_send_modbus_request` and receiving a response, which is handled via the function `app_poll_modbus`. Here is how they work - 

### Requesting via modbus
```
void app_send_modbus_request(pal_uart_t* uart, uint8_t slave_id, uint8_t func_code, uint16_t start_addr, uint16_t quantity)
```

We send over the uart port at which the RS485 chip is connected to, the slave ID of the device that we wish to talk to, the function code of the request that we are sending, the start address and the quantity of registers that needs to be read. 

We construct a tx frame of 8 bytes, 6 of which are actually being used. Slave_id, function code, start ID's higher byte, start ID's lower byte,  Quantity's higher byte, and Quantity's lower byte. 

The last 2 bytes have a crc that gets re-calculated at the end - don't worry about that. This frame is then sent to the mfm via the UART write call at the end. 
### Reading Modbus responses
```
void app_poll_modbus(pal_uart_t* uart, modbus_rtu_ctx_t* ctx, uint32_t current_time)
```

This is the response handler that takes care of the incoming data packet from the MFM. Here, we pass the uart instance, an object of a custom structure, and the timestamp at which the function has been called. 

The function's algorithm works as follows - 
1. Because of a silent time interval requirement that the MODBUS standard specifies, we have to introduce a delay or a wait in the reception.  (Might not be needed because of the async nature of our system. )
2. We read a total of 1 byte from the received MODBUS frame, and this is reconstructed in the context's rx_buffer. Then, we keep reading till the context's write index is beyond 5, since that is  where the MODBUS data lies in the frame. 
3. At rx_buffer[1], we check if the response function code is <something+0x80>, since that is the mark of an error frame. If not, we service the function codes 0x01 or oxo3. 
4. Once the function code is checked, the expected length of the frame is determined (either 5 bytes for an error frame, or the Data Length Code `ctx->rx_buf[2]` plus 5 bytes for standard function codes 03 or 04). The function waits until the context's write index (`ctx->idx`) reaches this `expected_len`.
5.  When the complete frame is captured, its integrity is verified by passing the buffer and expected length into `modbus_calc_crc`. A valid Modbus frame will yield a CRC result of 0.
6. If the CRC is valid, the payload is forwarded using `route_incoming_message`. For a standard response, it passes the context's source ID, the slave ID (`ctx->rx_buf[0]`), the payload length (`ctx->rx_buf[2]`), and a pointer to the actual data (`&ctx->rx_buf[3]`). If it is an error response, the slave ID is logically OR'd with `0x8000` to signal the error upstream.
7. After the frame is processed, the context index (`ctx->idx`) is reset to 0 so the system is ready to receive the next incoming Modbus frame.

# Log Storage via SD Card

## Semantics of the Files being maintained. 
We are writing binary files to the SD card, primarily to save space on the card itself. We have python scripts on the laptop that helps decode the binary files. Each file that is being written, has a size limit of 50MB. 

## Maintaining files on the SD Card
The crux of the approach -  **don't maintain the state separately; derive it from the state of the filesystem itself.**

Instead of having an `INDEX.TXT` file telling us where the logger left off, the logger simply looks at which log files already exist on the SD card. On mounting the card, the logger starts at `LOG_0001.BIN` and checks each possible filename sequentially. The first file that does not exist is treated as the next file to be written.

This creates what we can think of as a deliberate "gap" in the sequence.  On an empty card, the first file checked is `LOG_0001.BIN`. Since it doesn't exist, that becomes the next file to write.  If files `LOG_0001.BIN` through `LOG_0045.BIN` already exist, the first missing file is `LOG_0046.BIN`, so that becomes the next logging file.

The interesting case is rollover. Suppose the card contains files `LOG_0002.BIN` through `LOG_0100.BIN`, while `LOG_0001.BIN` has been removed. The search wraps around the sequence, finds the missing `LOG_0001.BIN`, and uses that as the next file. After identifying the missing file, the logger opens it for streaming and then deliberately deletes the _next_ file in the sequence.

This is what maintains the gap.

For example, if `LOG_0046.BIN` is the missing file, the logger opens `LOG_0046.BIN` and deletes `LOG_0047.BIN`. Once `LOG_0046.BIN` starts being written, the sequence now contains a single gap at `LOG_0047.BIN`. On the next mount, the search will eventually find that gap and select `LOG_0047.BIN` as the next file.

At the rollover boundary, the same mechanism wraps around. If `LOG_0100.BIN` is the file being created, the "next" file becomes `LOG_0001.BIN`. If `LOG_0001.BIN` is subsequently missing, it becomes the next target.

There is also a useful edge case here: a card that has been completely populated with 100 files. In that situation, the search finds no gaps at all, so the logger falls back to `LOG_0001.BIN`. It then opens that file and deletes `LOG_0002.BIN`, effectively creating the gap that the algorithm needs for the next cycle.

The major difference compared to Method 1 is that there is no additional state to maintain. The filesystem itself is the source of truth. If the user removes files manually, copies old log files onto the card, or deletes `INDEX.TXT` - well, there is no `INDEX.TXT` to worry about. The algorithm simply looks at what actually exists and works from there. 

The tradeoff is that we have exchanged storage efficiency for filesystem operations. Instead of immediately knowing which file to use, we potentially perform up to 100 `file_exists()` checks every time the SD card is mounted. With only 100 possible files, however, this is a very manageable cost. More importantly, the resulting state is self-healing: the logger does not depend on a separate piece of metadata remaining synchronized with the actual files on the card.