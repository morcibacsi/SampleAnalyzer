# Vehicle Area Network (VAN bus) Analyzer for Saleae USB logic analyzer

This plugin for [Saleae Logic][logic] allows you to analyze [VAN][van_network] bus packets found in cars made by PSA (Peugeot, Citroen)
VAN bus is pretty similar to CAN bus. In the application the **&#9679;** are marking the places where the sample was taken and the **&#10799;** are marking the E-Manchester bits (those must be ignored when decoding the data).

![logic analyzer](https://github.com/morcibacsi/VanAnalyzer/raw/master/docs/Logic_printscreen.png)

## Exporting

The analyzer exports to a simple text format with a timestamp, the channel and the bytes in hexadecimal. 
At the end of the packet the A stands for ACK required and N is for ACK not required. 
The two bytes before the A or N is the Frame Check Sequence (FCS).
The byte after the identifier is the command byte.

```
08.691351125000001 Ch:0: 5E4 C 20 1F 95 6A N
08.723351583333333 Ch:0: 8C4 C 8A 22 5A 81 5E A
08.736818458333332 Ch:0: 8D4 C 12 01 E8 2E A
08.799568666666666 Ch:0: 8C4 C 8A 21 40 3D 54 A
08.800854083333334 Ch:0: 4D4 E 85 0C 01 01 31 0E 3F 3F 3F 47 85 12 06 A
08.807504375000001 Ch:0: 554 E 81 D1 01 80 EE 02 00 04 FF FF A1 00 00 00 00 00 00 00 00 00 00 81 15 2E A
09.190060833333334 Ch:0: 5E4 C 20 1F 95 6A N
09.251457750000000 Ch:0: 8C4 C 8A 24 40 9B 32 A
09.252726958333334 Ch:0: 554 E 81 D1 01 80 EE 02 00 04 FF FF A1 00 00 00 00 00 00 00 00 00 00 81 15 2E A
09.686664166666667 Ch:0: 5E4 C 20 1F 95 6A N
09.806916458333333 Ch:0: 4D4 E 86 0C 01 01 31 0E 3F 3F 3F 47 86 55 E4 A
```

## Protocol

There is a pretty good description about the protocol on Graham Auld's page [here][graham]. I copied the key part here for a quick reference.

### Data Frames

VAN data is broadcast on the bus in discreete frames of data encoded using E-Manchester that include an Identifier to allow recievers to filter the data they are interested in. Frame marking is made through the use of E-Manchester violations

This coding of every 4 bits by 5 bits makes length information a little ambiguous unless you remark on the codin (CAN is even more of a pain through the use of data dependant bitstuffing to enable clock recovery). 
For this reason I like to refer to Time Slices at this level, a Time Slice (TS) is the time taken to transmit a dominant or recessive bit on the bus in real time. 
Hence 4 bits take 5TS to transmit due to the E-Manchester bit being appended. A 125kbps VAN bus results in 1TS=8uS.


|Field Name  | Length (TS)   | Purpose   |
|------------|:-------------:|--------   |
|Start of frame |   10       |Marks start of frame, provides edges for clock synchronisation always 0000111101     |
|Identifier     |   15       |Identifies the packet for recieve filtering, also provides priority in bus access arbitration, lower identifiers get higher priority bus access. 0x000 and 0xFFF are reserved    |
|Command        |   5        |These four bits, Extenstion (EXT), Request Acknowledge (RAK), Read/Write(R/W) & Remote Tramsmission Request(RTR) provide instruction to recievers. EXT is reserved and should be 1 (recessive), RAK set to 0 means that no acknowledge should be provided, R/W marks the frame as a read(1) or write(0) request, RTR set to 0 means the frame contains data, 1 means the frame contains no data and is a request to send.    |
|Data           |   0+       |0-28 bytes usually but potentially up to 224 bytes of data    |
|Frame Check Sequence   |   18  |This is the CRC-15 of the identifier, command and data fields used to ensure the integrity of recieved data    |
|End of Data    |   2        |This transmits a pair of zeros which commit an E-Manchester violation marking the end of transmitted data.    |
|Acknowledge    |   2        |Two recessive states, an acknowledge is given by one or more receivers transmitting a dominant state during the second state    |
|End of Frame   |   8        |8 successive recessive states (1s) which mark the end of the frame    |
|Inter-Frame Spacing    |   8   |Not strictly part of the frame but there must be at least 8TS of recessive state between each frame on the bus    |

### Example data

![example](https://github.com/morcibacsi/VanAnalyzer/raw/master/docs/vanex.png)

This example data was captured with a 'scope from a head unit removed from the car and decoded by hand.

Some example captures can be found in the SampleCaptures folder.

## Building

The libraries required to build a custom analyzer are stored in another git repository, located here:
[https://github.com/saleae/AnalyzerSDK](https://github.com/saleae/AnalyzerSDK)

Note - This repository contains a submodule. Be sure to include submodules when cloning, for example `git clone --recursive https://github.com/morcibacsi/VanAnalyzer.git`. If you download the repository from Github, the submodules are not included. In that case you will also need to download the AnalyzerSDK repository linked above and place the AnalyzerSDK folder inside of the SampleAnalyzer folder.

*Note: an additional submodule is used for debugging on Windows, see section on Windows debugging for more information.*

To build on Windows, open the visual studio project in the Visual Studio folder, and build. The Visual Studio solution has configurations for 32 bit and 64 bit builds. You will likely need to switch the configuration to 64 bit and build that in order to get the analyzer to load in the Windows software.

To build on Linux or OSX, run the build_analyzer.py script. The compiled libraries can be found in the newly created debug and release folders.

	python build_analyzer.py

To debug on Windows, please first review the section titled `Debugging an Analyzer with Visual Studio` in the included `doc/Analyzer SDK Setup.md` document.

Unfortunately, debugging is limited on Windows to using an older copy of the Saleae Logic software that does not support the latest hardware devices. Details are included in the above document.

## Installing

In the Developer tab in Logic preferences, specify the path for loading new plugins, then copy the built plugin into that location.

## TODO
- Better handling of finding the beginning of the first whole frame in the stream. (now sometimes it fails to decode the first frame, and sometimes it detects garbage as data)
- Find a way to mark the Frame Check Sequence bytes as FCS instead of DATA
- Implement GenerateSimulationData() method (now it has the default from the SampleAnalyzer)


[logic]: https://www.saleae.com/downloads
[graham]: http://graham.auld.me.uk/projects/vanbus/lineprotocol.html
[van_network]: https://en.wikipedia.org/wiki/Vehicle_Area_Network
