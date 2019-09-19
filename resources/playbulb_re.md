Android Bluetooth Debugging

- In developer options, turn on "Enable Bluetooth HCI snoop log". Will create a log at "/sdcard/btsnoop_hci.log"
- To download the file, "adb pull /sdcard/btsnoop_hci.log"
- To delete the file, "adb shell rm /sdcard/btsnoop_hci.log"

Open the file in Wireshark to see bluetooth packets
wireshark -r btsnoop_hci.log




App Functionality
    Rename playbulb: not sure if this does anything to the bulb
    Factory Reset
    Set Password to Device

    Shake  (off/on)
    Colour (on/off)
    Colour picker
    Saturation (scale)
    Colour

    Effects
        Candle
            Candle Effect (on/off)
            Colour switch (on/off)
            Colour (colour slider)
        Flashing
            Speed (slider)
            Colour switch (on/off)
            Colour (colour slider)
        Pulse
            Speed (slider)
            Set of colours
        Rainbow
            Speed (slider)
        Rainbow Fade
            Speed (slider)
                FFFB = `0x00ed00ff03002700`
                            ^         ^ - byte here is fade speed (fastest: 0x14, slowest: 0xaf)
                            | 3 hex bytes for RGB (eg ed00ff)
    Scenes

    Timer

    Security


## Bluetooth Notes

Pairing vs Bonding

Linux only: gatttool, hcitool

`echo -n "totes new" | od -A n -t x2 | sed 's/ //g' | head -n 1 | pbcopy` # get hex from text

http://rgb.to/

`sudo hcitool lescan` if on linux to get MAC address of candle
`gatttool -b AC:E6:4B:06:6F:CD -I` start gatt in interactive mode

Look for device advertising a service 

How to find the characteristics for a service on a BLE device?

How do we know which service a GATT read request is sent to?




#### Matching Actions to Captured Packets

* Clean out the btsnoop log on the android device.
* Start the MIPOW app
* Connect to the candle and start a timer
* Perfom an action and record the timer time, eg change bulb colour
* Repeat above step until you have performed actions you want with at least a few seconds between actions
* Open in wireshark, use Time column to correlate each action with bluetooth messages


## Playbulb Protocol

Advertises a service on FF02 with the following characteristics:
    * FFF8 = ? 
        Never any value
    * FFF9 = ?
        `0x000000ffffffff000000000000` with a solid blue 50% hue light, no effects on
    * FFFA = ? active effect?
        `0x00` with a solid blue light, no effects on
        `0x00` with a max speed red pulse
    * FFFB = ?
        When cycling through colours in Rainbow Fade, the value is something like `0x00 0064ff 0300 0a00` where the second part is the hex colour of the candle at that instant. This characteristic constantly changes in this mode. final block may be speed of colour change in Rainbow Fade mode.
        `0x00000000ff004000` when solid blue full hue. Changing hue value does not change this.
        `0x00ff000000001400` when in max speed red flashing. 2nd to last byte controls speed of flash. 00-FF. When on FF period is about 2s. 0F
        [rgb][rgb][effect byte][? byte]
        `0x00ff000001001400` when max speed red pulse. 2nd to last byte is speed of flash. 00-FF. 5s full cycle time when 00, 1s full cycle time when 01, 9s fct when 10, 2mins full cycle time on FF.
        `0x00ff000004000100` red candle effect on
        `0x00ff0000ff000100` red, no candle effect
        `0x0018ff0004000100` green, candle efffect
        `0x0018ff0003000100` cycle through colours. Not from app command.
        `0x0018ff0003000100` cycle very rapidly (flash) through colours. Not from app command.
        `0x00ffff0002001400` rainbow, max speed
        `0x0000ff000200ae00` rainbow, min speed. 2nd to last byte is speed. 00-FF. FF=2s on each colour, 80 = 1s per colour, 00 = strobe.
        `0x006e00ff03001400` rainbow fade, max speed
        `0x006e00ff01000300` single colour, fade in and out. more of a pop back into colour.
    * FFFC = candle colour - only if no effect enabled? or effect and colour.
        * `0xff000000` when white candle mode on
        * `0x00000080` when solid blue with no effects but some hue
        * first byte controls brightness? FF = dim colour, 00 = bright colour
        `0x7a000000` when max speed red flashing
        `0x7a000000` when max speed red pulse
    * FFFD = ?
        ? `0x77c52370` with a solid blue light, no effects on
          `0x77c52370` with red pulse max speed
    * FFFE = ?
        ? `0x04ffff04ffff04ffff04ffff0000` with a solid blue light, no effects on
    * FFFF = device name in utf 8

rainbow colour cycle order: 0000ff, ff00ff, ff0000, ffff00, 00ff00, 00ffff

What is the off command?

Battery Service
    * battery level (percentage of charge left) (0x64 = 100)

Opening a connected bulb in the app causes read requestst to handles:
    * 0x0019 with return `02 0a 20 09 00 05 00 04 00 0b 00 00 00 00`
    * 0x001f with return `02 0a 20 14 00 10 00 04 00 0b 50 4c 41 59 42 55 4c 42 20 43 41 4e 44 4c 45`
    * 0x0017 with return `02 0a 20 0d 00 09 00 04 00 0b 00 ff 00 23 03 00 0a 00`

Changing the colour to pure green in candle mode sends a write to handle 0x0019 with the value `0000ff00`

5 seconds later, we get:
`Sent Vendor Command 0x0157 (opcode 0xFD57)` and `Rcvd Command Complete (Vendor Command 0x0157 [opcode 0xFD57])` with value `04 0e 06 01 57 fd 00 00 01`
`Sent Vendor Command 0x0157 (opcode 0xFD57)` and `Rcvd Command Complete (Vendor Command 0x0157 [opcode 0xFD57])` with value `04 0e 07 01 57 fd 00 01 00 0f`
`Sent Vendor Command 0x0160 (opcode 0xFD60)` and `Rcvd Command Complete (Vendor Command 0x0160 [opcode 0xFD60])` with value `04 0e 04 01 60 fd 00`
No idea what the above 3 commands are.

Wireshark filter: btatt.opcode == 0xa || btatt.opcode == 0x0b || btatt.opcode == 0x52  || btatt.opcode == 0x1b

Can't read the GATT attributes while connected to the MIPOW app, so have to connect with app, make change, disconnet, connect with laptop bluetooth, then read attribute.


## References

### Bluetooth Capture
https://www.nowsecure.com/blog/2014/02/07/bluetooth-packet-capture-on-android-4-4/
http://rexstjohn.com/bluetooth-smart-ble-scanning-on-osx-xcode-6/

### Bluetooth LE
http://www.slideserve.com/Samuel/ble-101-bluetooth-low-energy
https://github.com/tigoe/BLEDocs/wiki/Introduction-to-Bluetooth-LE
https://github.com/liquidx/CoreBluetoothPeripheral - Sample for Bluetooth on OSX

### Playbulb
https://github.com/Phhere/Playbulb/blob/master/protocols/color.md
https://github.com/danielsmith-eu/playbulb-live
https://pdominique.wordpress.com/2015/01/02/hacking-playbulb-candles/
https://www.npmjs.com/package/playbulb
https://codelabs.developers.google.com/codelabs/candle-bluetooth/#2



## Cocoa Console App

`playlbulb --mode FLASH --speed 24 --colour 33ff55 --device-name slackbot`

All playbulbs have same serial and device name by default? 'BLT300' and 'PLAYBULB CANDLE'
All advertise a service at FF02. 

