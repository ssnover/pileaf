# Protocol Notes
Originally taken from a blog post here: https://kronoshacker.blogspot.com/2018/01/playing-with-aurora-led-panels.html

## Playing with the Aurora LED Panels
I recently got my hands on a bunch of Nanoleaf Light Panels for a decent price. To be honest, I have been thinking about buying some for some time now, but IMHO they are quite expensive at about 20 Euros per piece.
There is a nice Teardown on EEVblog, but it stops just where things get interesting from a hacker's point of view: the communication protocol you need if you want to control the tiles yourself. I decided to reverse engineer this protocol and to document it in this post.

### Physical Layer
Let's first look at the physical layer. Each tile is connected to it's neighbors via a 4 pin connector. The 4 Pins are:

1. Power Supply (24V)
2. Ground
3. Select (3.3V active high)
4. Comm (24V active low, open collector)

Pins 1, 2 and 4 are physically connected between the three connectors on the tile's edges. Pin 3 is of each edge is connected to the tile's microcontroller and can be controlled individually for each edge of the tile.

#### The Comm Pin
The Comm Pin is used for data communication between the tiles. It is a open collector bus which can be driven to low by any tile. The protocol must take care of collision avoidance. RS232 at 115200 Baud, 8N1 is used for the communication on this pin. Inactive level is 24V, Active level is 0V. The signal is pulled to 24V by the controller. The tiles drive the signal to GND via a open collector circuit.

#### The Select Pin
The Select Pin is used to enumerate the tiles and to discover topology. Usually the Select Pin is a Ground level, but it can be pulled to 3.3V for each edge of a tile separately if requested by the controller. 

### Protocol on Comm Pin
The protocol on the comm pin is not too complicated. The first byte in each command is always the command type. The second byte is always the short id of the addressed tile (0xFF addresses all tiles). The rest of the bytes are the optional command payload. Some commands are followed by two bytes checksum (CRC16 CCIT). Some commands trigger a response by the addressed tile which follows immediately after the command. Some responses also are followed by a CRC16.

#### Short and long tile IDs
Each tile has a unique 8 Byte long ID that uniquely identifies this specific tile. This long ID is used in the enumeration phase and for topology discovery. In the enumeration phase each tile is assigned a 1 Byte short ID that from then on used to address the tile.

### Commands known so far

#### 0x01 - Assign short ID by long ID
This command is used to assign a tile a short ID if it's long ID is known. If the long ID of the tile matches the long ID in the message, the tile sets i't short ID to the short ID in the message (Byte 11) and responds. The message looks like this:

| Byte | Explanation |
| ---  | --- |
| 1 | Command Byte (0x01) |
| 2 | Short ID byte (always 0xFF since addressing is done via Long ID)
| 3-10 | Long ID of the addressed tile |
| 11 | Short ID to be assigned to this tile |
| 12-13 | CRC16 of bytes 1-11 |

The tile with the Long ID matching the Long ID in the message responds with 80 00 00 if present.

#### 0x02 - Set Edge Pin
This command is used to set the Edge Pins of the addressed tile. The only payload byte is a bitmask of the levels to be set on the three Edge Pins (0x01 / 0x02 / 0x04). There is no CRC for this command.

| Byte | Explanation |
| ---  | --- |
| 1 | Command Byte (0x02) |
| 2 | Short ID Byte |
| 3 | Edge Pin configuration |

The tile with the Short ID matching the Short ID in the message sets it's Edge Pins according to the message and responds with 80 00 00 if present.

#### 0x03 - Read Long ID if selected
This command is used to read the long ID from a tile that is selected by pulling one of it's Edge Pins high from a neighbor tile. A tile only responds to this message if one of it's Edge Pins is pulled high from the outside.

| Byte | Explanation |
| ---  | --- |
| 1 | Command Byte (0x03) |
| 2 | Short ID byte (always 0xFF since addressing is done via Edge Pin) |

The tile responds with the following sequence:

| Byte | Explanation |
| ---  | --- |
| 1 | Response Byte (0x83) |
| 2-3 | Unknown |
| 4-11 | Long ID of the responding tile |
| 12-13 | CRC16 of bytes 1-11 |

#### 0x08 - Fade Tile to Color
This command is used to set the Color of a specific tile or all tiles. A fade time can be given. There is no CRC and no response for this command.

| Byte | Explanation |
| ---  | --- |
1 | Command Byte (0x08)
2 | Short ID byte (or 0xFF for all tiles)
3 | Red color value from 0x00 to 0xFF
4 | Green color value from 0x00 to 0xFF
5 | Blue color value from 0x00 to 0xFF
6 | White color value from 0x00 to 0xFF
7-8 | Fade time MSB first 0x0000 to 0xFFFF

#### 0x09 - Call for unconfigured tiles
This command is used to check if new tiles have been added to the network. Only tiles which have not been assigned a short ID will respond to this command with 80 00 00. There is no CRC for command or response.

| Byte | Explanation |
| ---  | --- |
1 | Command Byte (0x09)
2 | Short ID byte (always 0xFF)

#### 0x0A - Check tile presence
This command is used to check if a specific tiles has been removed from the network. The addressed tile will respond to the command with 80 00 00. There is no CRC for command or response.

| Byte | Explanation |
| ---  | --- |
1 | Command Byte (0x09)
2 | Short ID byte

#### 0x0B - Set Brightness
This command is used to set the brightness of a specific tile or all tiles.

| Byte | Explanation |
| ---  | --- |
1 | Command Byte (0x0B)
2 | Short ID byte
3 | Brightness Value from 0x00 to 0xFF

#### 0x0C - Read tile info
This command is used to read some version information from the tile. The version numbers are encoded in ASCII, unused bytes are set to 0x00. The addressed tile responds with its version information and the CRC.

| Byte | Explanation |
| ---  | --- |
1 | Command Byte (0x0C)
2 | Short ID byte

The tile responds with the following sequence:

| Byte | Explanation |
| ---  | --- |
1 | Response Byte (0x8C)
2-20 | Version Information
21-22 | CRC16 of bytes 1-20