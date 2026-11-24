[[RhyGen]]

![[i2c_pal.excalidraw]]

Buckle Up. This is going to be complex. 

We have a state machine going on here, that simply keeps track of what is going on within the I2C bus, and keeping checks and balances as we go on about the transmission. This is documented in the attached drawing. The point here is to understand the end-to-end flow of the whole operation. 

- **Step 1, `submit`:** it **validates the descriptor first**. It returns `PAL_I2C_ERR_PARAM` for an address above `0x7F`, for both lengths being 0, or for a NULL buffer whose length is non-zero. It returns `PAL_I2C_ERR_FULL` if the push fails. In both cases `done()` will never be called. Also, `start_next` does nothing unless the state is `XFER_IDLE`. If a transaction is already in flight, the new one waits in the queue until `process()` gets to it. 
- **Step 2, `start_next`:**
    - It **checks for `XFER_IDLE` before popping**.
    - It **replaces a `timeout_ms` of 0** with `PAL_I2C_DEFAULT_TIMEOUT_MS`.
    - `try_start` **only calls `start_hw` if the bus BUSY flag is clear**. If the flag is set, the state stays `XFER_WAIT_BUS`, and each `process()` call either retries or ends the transaction with a timeout. This is what keeps the HAL's 25 ms internal busy-wait from ever running.
    - On "what keeps it async": the ringbuffer only holds waiting work. What makes it non-blocking is that the `*_IT` HAL calls return immediately and completion is detected later by `process()`. You could have async with a queue of one.
- **Step 3, `start_hw`:**
    - It **sets `state` to `XFER_WRITE` or `XFER_READ` before calling the HAL**. The HAL enables the interrupts as the last thing it does, so the transfer can finish, and the ISR can check the state, before the HAL call even returns.
    - On the write path, the frame option depends on `rx_len`: `I2C_FIRST_FRAME` if a read follows (no STOP), and `I2C_FIRST_AND_LAST_FRAME` otherwise.
    - If the HAL returns anything other than `HAL_OK`, `start_hw` sets `result = PAL_I2C_ERR_BUS` and `state = XFER_COMPLETE`. That's safe because the HAL hasn't enabled the interrupts, so no ISR can race with it. 
    
    - 3.1 In `HAL_I2C_MasterTxCpltCallback`, we return straight away if the state is not `XFER_WRITE`. Otherwise the write half has finished. If the transaction also has a read (`rx_len > 0`), the write was sent with `I2C_FIRST_FRAME`, so no STOP was generated and we still own the bus. We set the state to `XFER_READ` and call `HAL_I2C_Master_Seq_Receive_IT` with `I2C_LAST_FRAME`. Because the direction changes, the HAL issues a repeated START, reads `rx_len` bytes and generates STOP after the last one. If the transaction is write-only, the HAL has already generated STOP and we set the state to `XFER_COMPLETE`.
      
    - 3.2 In `HAL_I2C_MasterRxCpltCallback`, we return straight away if the state is not `XFER_READ`. Otherwise all bytes are in the caller's `rx` buffer and STOP has been sent, so we set the state to `XFER_COMPLETE`. 
      
    - 3.3 In `HAL_I2C_ErrorCallback`, a pure acknowledge failure becomes `PAL_I2C_ERR_NACK`, and any other error becomes `PAL_I2C_ERR_BUS`. The state goes to `XFER_COMPLETE`. 
      
    - 3.4 None of the callbacks call application code. `process()` sees `XFER_COMPLETE` on its next pass, runs bus recovery if the result was a timeout or bus error, returns the state to `XFER_IDLE`, calls the transaction's `done()` callback with the result, and starts the next queued transaction.

--- 
## End To End Flow via the PAL
Apart from the BNO app, let's trace a random device that I wish to talk to via the I2C pal. 
Say that the device is at ID 0x30, and I wish to request 4 bytes of data from that device. I want the complete flow of this process, how shallI construct the transaction descriptor, which functions shall be called, when will the HAL callbacks fire, all end to end.

Assume that there are no timeouts or bus errors that happen along the way. 

I believe that upon initializing a xfer packet with the (correct address<<1), a buffer for the tx data to be stored at, setting the length of data to be transmitted as 2 (1st byte shall have the address and R/W bit, second byte shall have the number of bytes I am requesting), a pointer to the rx_buffer that I have setup for me to store the incoming data, setting the rx_len as 4, having the timeout_ms as 0, and the rest of the members initialized, I shall first call the submit function. Then, the process function shall be called in my loop, till I see something in the done() function that I have setup. 

Under the hood, the submit() shall validate the params, push transfer descriptor to the ringbuffer and call the start_next function. 

Start_next() checks if the state is idle or not, pops the ringbuffer, checks and sets the active timeout to default if it is set as 0, records the starting tick count, moves the state to wait_bus, and calls try start. Try_start() simply acts as a guard against the busy flag, and calls start_hw. 

Start_hw)_ shall check the tx_len, find out that we have set it as 2, and shall call the HAL_I2C_Master_Seq_Transmit_IT function. In that call, since the rx_len is set as 4, we shall be sending the HAL_I2C_Master_Seq_Transmit_IT function with XFerOptions as I2C_FIRST_FRAME. 

The corresponding interrupt shall trigger HAL_I2C_MasterTxCpltCallback ONLY WHEN WE HAVE TRANSMITTED ALL THE BYTES THAT WE HAVE SPECIFIED, where we shall see that the rx_len is 4, and we shall call the HAL_I2C_Master_Seq_Receive_IT function, with XFerOptions as I2C_Last_Frame. This triggers the HAL_I2C_MasterRxCpltCallback, ONLY WHEN WE HAVE RECEIVED ALL THE BYTES THAT WE HAVE SPECIFIED. Here we transition the state to XFER_COMPLETE. 

In the next call to process(), we see the status as PAL_I2C_OK, we transition the state to Idle, and fire the done() application callback if it has been declared, and then call the start_next for sending over the next transaction. 