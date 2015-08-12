##### 10 Aug 2015 11h -   Messed around with GEOMESS.
Messed around with GEOMESS, and examined a bit how it's structured and works. Will use GEOMESS with an MTU of 127 to test the 6LoWPAN adaption layer.

##### 10 Aug 2015 16h -   Begun on the 6LoWPAN adaption layer skeleton [WIP].
Started defining the skeleton of the adaption layer. The 6LoWPAN adaption layer will exist next to (and just like) the PPP adaption layer. For a 6LoWPAN-capable device, the abstraction layer will provide the structure to make a picoTCP-compatible device (pico_device). When the device is initialised it will be added to the picoTCP-device tree and frames are passed on from the IPv6-layer to the device layer, which is the 6LoWPAN adaption layer.

##### 11 Aug 2015 00h -   Defined 802\.15\.4 MAC frame structure.
A MAC layer is needed in order to do 802.15.4-communications. This MAC layer is responsible for generating MAC frames that can be passed on to the PHY (device-driver). This MAC-layer will be part (as a sublayer) of the 6LoWPAN adaption layer.
So I started defining the framing format of the 802.15.4 MAC frames.

##### 11 Aug 2015 18h -   6LoWPAN Development Process.
So the goal of this project is to develop a 6LoWPAN adaption layer for picoTCP. In the end 6LoWPAN-capable devices should be able to communicate with each other by sending IPv6-frames over IEEE802.15.4-RF, all through the use of picoTCP.
For explanatory reasons and to give myself a grip during development, I will give a small overview on the development process and what tasks need to be done to realise this project.
So the entire project exists out of 4 key issues, which build up incrementally. This 4 key issues are:

- **802.15.4 Communication**
- **Simple IPv6 over 802.15.4**
- **Full 6LoWPAN**
- **6LoWPAN Neighbour Discovery**

For each of these issues a couple of things need to be finished/done in order solve the key issue itself. An overview of each item that needs to be done before I can proceed to the next item to solve a key issue is given below:

###### 802.15.4 Communication
1. ***Setting up a PAN  -***
In order to communicate over 802.15.4 someone has to set up a PAN. This is done by a PAN coördinator. The tasks of this PAN coördinator involves setting a PAN identifier, finding a suitable radio frequency (channel) and assigning a 16-bit short address to himself (0x0000).
These tasks are done in a device-driver, but before writing a universal device-driver this can be done hardcoded.
2. ***Joining a PAN  -***
After a PAN coördinator has set up a PAN other hosts can join that PAN. This can be done by setting the radio-frequency to the same channel as that of the PAN and setting the PAN ID to the ID of the PAN. These tasks are done in a device-driver, but before writing a universal device-driver this can be done hardcoded.
3. ***Sending and receiving data over PAN  -***
When 2 devices are in the same PAN they can communicate with each other. Therefore, a MAC layer is needed for addressing. Based on the MAC header, the 802.15.4-transceivers will apply address filtering mechanism and discard frames that aren't directed to the node. This communication needs to be examined to see if the 802.15.4 communication runs smoothly before the final item can be finished.
4. ***Pouring the 802.15.4 MAC framing and communication in a MAC-layer  -***
This MAC layer provides the mechanisms to encapsulate data in a 802.15.4-MAC frame which is send over the air by the PHY. Add first, this can be done hardcoded, but this can (probably) be integrated in the 6LoWPAN-adaption layer described below.

###### Simple IPv6 over 802.15.4
1. ***Send small IPv6-packet over 802.15.4  -***
This is the humble beginning of the adaption layer. The adaption layer receives a small enough packet from the network-layer (IPv6) so that fragmentation and compression isn't needed. This IPv6-packet is encapsulated in a 802.15.4-MAC frame which is passed on to the PHY and sent to a remote host. The received frame is decapsulated and the IPv6-packet is examined. If the IPv6-packet is received correctly, I can proceed to the next key issue.

###### Full 6LoWPAN adaption layer
1. ***Add header compression  -***
The tiny adaption layer from previous issue is extended with header compression. This can be done with several compression schemes.
2. ***Add fragmentation for too big packets  -***
The tiny adaption layer from previous issue is extended with fragmentation. When an entire IPv6 packet passed by the network layer doesn't fit into a single 802.15.4-MAC frame, the IPv6-packet is fragmented and sent seperately over the air. At the receiving host the packet is reassembled and examined for correctness.
3. ***Add Mesh routing  -***
The tiny adaption layer from previous issue is extended with Frame Delivery in a Link Layer Mesh. This is done with the 6LoWPAN Mesh header and is also known as mesh-under network.
4. ***Add broadcast messaging and duplicate packet suppression  -***
The tiny adaption layer from previous issue is extended with LoWPAN Broadcast with the help of the 6LoWPAN Broadcast Header. Duplicate Packet Suppression is herefore a requirement.

###### 6LoWPAN Neighbour Discovery
1. ***WIP***

##### 11 Aug 2015 22h -   Modified GEOMESS with MTU 127.
Added a new device-structure to the geomess-library. This device represents the 6LoWPAN adaption layer. Currently the simulation is done in the adaption layer itself but I will update this tommorrow to make use of a device-driver. The device-driver will be responsible for transmitting and receiving on/from the Geomesh-network, while the adaption layer (pico_dev_sixlowpan) whill be responsible for the right formatting and MAC encapsulation to pass on to the device-driver. The development of the adaption layer will occur in pico_dev_sixlowpan.

##### 12 Aug 2015 14h -   Differentiated 6LoWPAN adaption layer from device-driver.
Today, I differentiated the 6LoWPAN adaption layer (pico_dev_sixlowpan) from the application-specific device-driver. Now, the 6LoWPAN-header provides a generic device-driver structure (radio_t). Every device-driver must be compatible with this structure. The 6LoWPAN adaption layer can then communicate with the device-driver. So here is how it works with a Geomess-capable device-driver:

**pico_dev_sixlowpan** is a custom pico_device-structure with internal structure as follows:

```C
struct pico_device_sixlowpan {
	struct pico_device dev;
	radio_t *radio;
};
```

*radio_t* is defined in the header of *pico_dev_sixlowpan* (pico_dev_sixlowpan.h) like this:

````C
typedef struct RADIO {
	int (*transmit)(struct RADIO *radio, const void *buf, int len);
	int (*receive)(struct RADIO *radio, void *buf, int len);
} radio_t;
```

For now, the radio-structure only defines a transmit- and receive-function. So, the application-developer can now provide a device-specific driver by instantiating a struct of type '*radio_t*' an providing all the given callback-functions.
Then, he can assign that specific radio-instance to the 6LoWPAN-adaption layer by making use of the function:

```C
int pico_sixlowpan_set_radio(struct pico_device *dev, radio_t *radio);
```

Now the 6LoWPAN adaption layer has a direct interface with the device driver. In this case, the device-driver (radio_driver) transmits and receives data on and from the Geomess-network. This is done by making the radio_t structure itself a custom one, with a Geomess-connection as a member like so:

```
typedef struct gm_radio
{
	radio_t radio;
	char *sock;
	GEOMESS conn;
}
gm_radio_t;
```

So when the 6LoWPAN adaption-layer calls 'transmit()' a radio-instance needs to be given. The transmit-function in the device-driver will then transmit the data on its own specific Geomess-connection.

**[NOTE]**: The seperate device-driver isn't really a requirement, the transmit- and receive-function could also be defined in the main-file, but seperating it gives me a nice overview of the application-structure and allows me to do some more simulation of the device itself.
