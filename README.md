# cpe2usb: Communicate with the Circuit Playground Express over USB!
This guide walks through how to communicate with the Circuit Playground Express over a USB connection. 
The guide was successfully tested on a 2018 Macbook Pro.

## Dependencies

Install the pyserial module on the host machine connecting to the CPE: https://pyserial.readthedocs.io/en/latest/pyserial.html

## Basic sending and receiving. 
The following code snippets show how to send a message to the CPE and "echo" it back to the host:

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

Note: `serial_bytes_available` is only true when there are bytes available to read on the serial communication channel (USB). You can read more about it in the [docs](https://circuitpython.readthedocs.io/en/latest/shared-bindings/supervisor/Runtime.html?highlight=serial_bytes#supervisor.Runtime.runtime.serial_bytes_available). 

On the *host* (computer that's connected over USB to the CPE), add the following `usb_send.py`:

Before running the file, determine the CPE's serial port. Follow 
[these instructions](https://learn.adafruit.com/welcome-to-circuitpython/advanced-serial-console-on-mac-and-linux#whats-the-port-15-1)
for linux and mac. 

You can type the following command on mac:
`ls /dev/tty.*` or for linux `ls /dev/tty.*`
to find the relevant port (the commands are essentially the same, but each platform uses slightly different naming conventions). On linux the port name is likely to look like `ttyACM0` (or some other number) 
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

# Streaming CPE sensor data over USB

Perhaps we wish to share temperature data from the CPE with the host. We can stream data using a similar method to that above.

Note that as communication is only going in one direction, we do not need to run the `serial_read()` function.
On *CPE*:

`code.py`
```python
import time
from adafruit_circuitplayground import cp


while(True):
    print("Temperature C:", cp.temperature)
    time.sleep(1)
```

On *Host*:

`usb_read.py`
```python
import serial

ser = serial.Serial(
             '/dev/tty<USB NAME>',
             baudrate=115200,
             timeout=0.01)

while(True):
    x = ser.readlines()
    if len(x) > 0:
        print("received: {}".format(x))
 ```

To run, add `code.py` to the CPE, update `USB NAME` in `usb_read.py` and then run it on the host system. The output should look like:
```
received: ['Temperature C: 25.5289\r\n']
received: ['Temperature C: 25.5289\r\n']
received: ['Temperature C: 25.551\r\n']
```
Yes, my apartment is nice and toasty 🔥

# Communication is a 2 way street 🛣️

The following is a more complex example which uses three-step protocol to communicate from CPE to host then back to CPE.

The desired behavior is as follows:
- When button A is pressed
- Enter a new color for into host's command prompt
- Change color on CPE to desired color

To do this, we will need to:
1. Send a button_message from CPE to host when button is pressed, then *wait* for a *color_message*
2. Host will *wait* for button_message, then query user for color, then send a *color_message* to CPE
3. The CPE, on receiving a *color_message* will change all LED colors

With this in mind, read the following code:

`code.py`
```python
import time
import supervisor
from adafruit_circuitplayground import cp

wait_color = False

while(True):
    if cp.button_a and not wait_color:
        print("button_message")
        wait_color = True

    if wait_color and supervisor.runtime.serial_bytes_available:
        value = input()
        rgb = value.strip().split(',')
        if rgb and len(rgb) == 3:
            r = int(rgb[0])
            g = int(rgb[1])
            b = int(rgb[2])
            cp.pixels.fill((r, g, b))
            wait_color = False
```

`color_host.py`

```python
import serial

ser = serial.Serial(
             '/dev/tty.usbmodem14301',
             baudrate=115200,
             timeout=0.01)

while(True):
    x = ser.readlines()
    if len(x) > 0:
        print("received: {}".format(x))
        in_msg = str(x[0].strip())
        print("parsed message: ", in_msg)
        if "button_message" in in_msg:
            r = input('red    (0-255): ')
            g = input('green  (0-255): ')
            b = input('blue   (0-255): ')
            out_msg = ','.join([r, g, b]) + '\r\n'
            ser.write(bytes(out_msg, 'UTF-8'))
```
To run, add `code.py` to the CPE (be sure the code updates), update `USB NAME` in `color_host.py` and then run it on the host system. 

The device should work as follows:
1. *Press Button A* on the CPE

2. The following should appear in the terminal where `color_host` is running:
```bash
received: [b'button_message\r\n']
parsed message:  b'button_message'
red    (0-255):
```

3. After the colon, enter a 'red' value for the LED (in the range 0-255) then press Return. Repeat this step for blue and green values. When you press return for the 'blue' value, the RGB values will be transmited to the CPE, and change the colors of all LEDs. 

Your entire terminal read out should look as follows:

```bash
received: [b'button_message\r\n']
parsed message:  b'button_message'
red    (0-255): 4
green  (0-255): 0
blue   (0-255): 1
received: [b'4,0,1\r\n']
parsed message:  b'4,0,1'
```

To try another color sequence, press button A, and repeat!

We leave analysis of the above code as an exercise for the reader. Happy hacking!

Thanks to [this](https://stackoverflow.com/questions/48922189/receive-data-from-host-computer-using-circuit-python-on-circuit-playground-expre) Stack Overflow post for the general awareness of this approach.
