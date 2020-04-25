# cpe2usb: Communicate with the Circuit Playground Express over USB!
This guide walks through how to communicate with the Circuit Playground Express over a USB connection. 
The guide was successfully tested on a 2018 Macbook Pro.

## Dependencies

Install the pyserial module on the host machine connecting to the CPE: https://pyserial.readthedocs.io/en/latest/pyserial.html

## Basic sending and receiving. The following code snippets show how to send a message to the CPE and "echo" it back to the host:

On the *CPE* add the following `code.py`:

```python
import supervisor

def serial_read():
   if supervisor.runtime.serial_bytes_available:
       value = input()
       print(value)

while(True):
   serial_read()
```

On the *host* (computer that's connected over USB to the CPE), add the following `usb_send.py`:

Before running the file, determine the CPE's serial port. Follow 
[these instructions](https://learn.adafruit.com/welcome-to-circuitpython/advanced-serial-console-on-mac-and-linux#whats-the-port-15-1)
for linux and mac. 

You can type the following command on linux/mac:
`ls /dev/tty.*`
to find the relevant port. On linux it's likely to look like `ttyACM0` (or some other number) 
and on mac, something like `tty.usbmodem14213`. Fill in the correct name below, where it says `<USB NAME>`

```python
import serial

ser = serial.Serial(
             '/dev/tty<USB NAME>',
             baudrate=115200,
             timeout=0.01)

ser.write(b'HELLO from CircuitPython\n')
x = ser.readlines()
print("received: {}".format(x))
```

Once `code.py` is loaded onto the CPE, run `usb_send.py` on your host system. 
It will send the phrase `"Hello from CircuitPython"` to the CPE, which will in-turn 
return it back to be printed. 
