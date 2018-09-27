# SWO Parser
This python script is used to parse ITM trace packets from the SWO pin on the
STM32 using OpenOCD. It is written for python 3, but shouldn't be too hard to
port to python 2 if you're one of those people. It communicates with OpenOCD
through the Tcl server (at localhost:6666).

## OpenOCD Settings
To use this script, first you must add some flags when you start up OpenOCD
(or add these to your startup cfg file for OpenOCD).

```
openocd.exe
-f board/st_nucleo_f7.cfg
-c "tpiu config internal - uart off 96000000"
-c "itm ports on"
```

The first line is because I'm using a nucleo_f7 demo board, substitute your
own cfg here. The line option tells OpenOCD to enable ITM tracing on the
target, tells it to send the data out on the Tcl server, and tells OpenOCD the
clock rate of the target. It's important that you replace the 96000000 number
with the actual clock rate for your target. The last line activates all 32 of
the ITM trace channels; it's optional if you just want to use channel 0 (which
is the channel used by ITM_SendChar).

## STM32 Settings
You shouldn't need to configure anything on the target. This fact took me
**forever** to figure out. When the debugger starts, it automatically sets the
proper registers to the right values. One nice side effect of this is that, if
the debugger is not attached to the target, your code will skip over sending
the trace messages. This is really nice since ITM_SendChar is blocking (i.e.,
the code stops until every byte is sent out on the SWO pin).

To send some data in your code, use ITM_SendChar. You can easily redirect
printf over this channel using that. Keep in mind though that **Messages will
not be displayed on the console until you send a newline character, '\n'**.
This is a result of the way I decided to implement things in this python
script. ITM_SendChar sends messages out on channel 0, but it's simple enough
to write your own version of it that can choose which channel to send data. You
could use this to filter types of messages (e.g, info, warnings, and errors).

## Running the Python Script
After starting up OpenOCD, open a terminal in the directory where you placed
swo_parser.py and type

```
python swo_parser.py
```

By default, messages will be parsed from channels 0, 1, and 2. Messages from
channels 1 and 2 will be prepended with "WARNING: " and "ERROR: " respectively.
You can configure these however you like. The following lines of the script
control the configuration:

```
# Create a stream manager and add three streams
streams = StreamManager()
streams.add_stream(Stream(0, '', tcl_socket))
streams.add_stream(Stream(1, 'WARNING: ', tcl_socket))
streams.add_stream(Stream(2, 'ERROR: ', tcl_socket))
```

The third parameter in the Stream constuctor (tcl_socket) is optional. Leave
it out if you don't want that message to be echoed to the GDB console.

