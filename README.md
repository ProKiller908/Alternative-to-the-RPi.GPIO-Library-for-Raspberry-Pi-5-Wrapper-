# Alternative-to-the-RPi.GPIO-Library-for-Raspberry-Pi-5-Wrapper-
If you are moving from and older Raspberry Pi (1 - 4) and have issues with the new Pi 5, with the Rpi.GPIO library then read the description!
You may had issues where the RPi.GPIO library did not work on the Pi 5, simple (MAY NOT 100%, AT YOUR OWN RISK! (WILL NOT DAMAGE DEVICE)) way, where I have provided a simple file and you must place it in the folder where the program that you use looks at executable, in my case Thonny, and also you must install libgpio, go to Terminal and isntall it first.
Once all of that is completed in your code just instead of wring "import RPi.GPIO" just write "import RPi_GPIO"!
