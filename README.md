# Alternative-to-the-RPi.GPIO-Library-for-Raspberry-Pi-5-Wrapper-
If you are moving from and older Raspberry Pi (1 - 4) and have issues with the new Pi 5, with the Rpi.GPIO library then read the description!
You may had issues where the RPi.GPIO library did not work on the Pi 5, simple (MAY NOT 100%, AT YOUR OWN RISK! (WILL NOT DAMAGE DEVICE)) way, where I have provided a simple file and you must place it in the folder where the program that you use looks at executable and name it RPi_GPIO.py, in my case Thonny, and also you must install libgpio, go to Terminal and isntall it first.
Once all of that is completed in your code just instead of wring "import RPi.GPIO" just write "import RPi_GPIO"!



Here is the code rename it RPi_GPIO.py later!

#!/usr/bin/env python3
"""
RPi_GPIO.py - A wrapper module that mimics the RPi.GPIO library using libgpiod.
This version includes support for both BCM and BOARD numbering schemes.
"""

import gpiod
import time

# Numbering mode constants
BCM = 'BCM'
BOARD = 'BOARD'

# GPIO direction and level constants
IN = 0
OUT = 1

HIGH = 1
LOW = 0

# Pull-up/down constants (for compatibility; not actively implemented)
PUD_OFF = 0
PUD_UP = 1
PUD_DOWN = 2

# Mapping from BOARD (physical) pin numbers to BCM GPIO numbers
_board_to_bcm = {
    # Left side of the header (physical pins 1–26)
    3: 2,    # SDA1
    5: 3,    # SCL1
    7: 4,
    8: 14,   # TXD0
    10: 15,  # RXD0
    11: 17,
    12: 18,
    13: 27,
    15: 22,
    16: 23,
    18: 24,
    19: 10,  # SPI_MOSI
    21: 9,   # SPI_MISO
    22: 25,
    23: 11,  # SPI_CLK
    24: 8,   # SPI_CE0_N
    26: 7,   # SPI_CE1_N

    # Right side of the header (physical pins 27–40)
    29: 5,
    31: 6,
    32: 12,
    33: 13,
    35: 19,
    36: 16,
    37: 26,
    38: 20,
    40: 21
}

# Internal variables
_mode = None
_chip = None
_channels = {}  # This will map BCM pin numbers to gpiod.Line objects

def setwarnings(flag):
    pass  # This simple wrapper does not implement warnings.

def setmode(mode):
    global _mode
    if mode not in (BCM, BOARD):
        raise ValueError("Invalid mode. Use BCM or BOARD.")
    _mode = mode

def init_chip():
    global _chip
    if _chip is None:
        try:
            _chip = gpiod.Chip('gpiochip0')
        except Exception as e:
            raise RuntimeError(f"Failed to open gpiochip0: {e}")

def _translate_pin(pin):
    """Convert a pin number to a BCM number based on the current mode."""
    if _mode == BCM:
        return pin
    elif _mode == BOARD:
        if pin not in _board_to_bcm:
            raise ValueError(f"Invalid BOARD pin: {pin}")
        return _board_to_bcm[pin]
    else:
        raise RuntimeError("GPIO mode not set. Use setmode(BCM) or setmode(BOARD).")

def setup(pin, direction, initial=None, pull_up_down=None):
    init_chip()
    bcm_pin = _translate_pin(pin)

    # Release the pin if it's already set up
    if bcm_pin in _channels:
        try:
            _channels[bcm_pin].release()
        except Exception:
            pass
        del _channels[bcm_pin]

    # Determine the request type based on intended direction
    req_type = gpiod.LINE_REQ_DIR_OUT if direction == OUT else gpiod.LINE_REQ_DIR_IN

    try:
        line = _chip.get_line(bcm_pin)
    except Exception as e:
        raise RuntimeError(f"Error getting line for pin {bcm_pin}: {e}")

    # Increase retry attempts if the line is busy
    max_attempts = 10
    attempt = 0
    while attempt < max_attempts:
        try:
            line.request(consumer="RPi_GPIO_wrapper", type=req_type)
            if direction == OUT and initial is not None:
                line.set_value(initial)
            break
        except OSError as e:
            if hasattr(e, "errno") and e.errno == 16:
                print(f"Pin {bcm_pin} busy, attempt {attempt+1}/{max_attempts}")
                time.sleep(0.05)
                attempt += 1
            else:
                raise RuntimeError(f"Error requesting pin {bcm_pin}: {e}")
    else:
        raise RuntimeError(f"Error requesting pin {bcm_pin}: Resource busy after multiple attempts")

    _channels[bcm_pin] = line

def output(pin, value):
    bcm_pin = _translate_pin(pin)
    if bcm_pin not in _channels:
        raise RuntimeError(f"Pin {bcm_pin} is not set up.")
    try:
        _channels[bcm_pin].set_value(value)
    except Exception as e:
        raise RuntimeError(f"Error writing to pin {bcm_pin}: {e}")

def input(pin):
    bcm_pin = _translate_pin(pin)
    if bcm_pin not in _channels:
        raise RuntimeError(f"Pin {bcm_pin} is not set up.")
    try:
        return _channels[bcm_pin].get_value()
    except Exception as e:
        raise RuntimeError(f"Error reading from pin {bcm_pin}: {e}")

def cleanup():
    global _channels
    for pin, line in list(_channels.items()):
        try:
            line.release()
        except Exception as e:
            print(f"Warning: Error releasing pin {pin}: {e}")
    _channels.clear()

