##### 10 Aug 2015 11h -   Messed around with GEOMESS.
Messed around with GEOMESS, and examined a bit how it's structured and works. Will use GEOMESS with an MTU of 127 to test the 6LoWPAN adaption layer.

##### 10 Aug 2015 16h -   Begun on the 6LoWPAN adaption layer skeleton [WIP].
Started defining the skeleton of the adaption layer. The 6LoWPAN adaption layer will exist next to (and just like) the PPP adaption layer. For a 6LoWPAN-capable device, the abstraction layer will provide the structure to make a picoTCP-compatible device (pico_device). When the device is initialised it will be added to the picoTCP-device tree and frames are passed on from the IPv6-layer to the device layer, which is the 6LoWPAN adaption layer.

##### 11 Aug 2015 00h -   Defined 802\.15\.4 MAC frame structure.
A MAC layer is needed in order to do 802.15.4-communications. This MAC layer is responsible for generating MAC frames that can be passed on to the PHY (device-driver). This MAC-layer will be part (as a sublayer) of the 6LoWPAN adaption layer.
So I started defining the framing format of the 802.15.4 MAC frames.
