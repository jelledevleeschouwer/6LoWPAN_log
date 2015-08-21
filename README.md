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

```C
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

##### 12 Aug 2015 15h -   Working repository.
All my code can be found in my fork of daniele's Geomess repository on: https://github.com/jelledevleeschouwer/geomess
The working directory is 'apps/IEEE802154'

##### 12 Aug 2015 20h -   Tested hardcoded 802.15.4-communication over Geomess.
Tried to send some hardcoded 802.15.4-MAC frames through *pico_dev_sixlowpan* over the Geomess-network, with sucess. Took some raw data buffer of a 6LoWPAN sample capture and used dev-send to send over Geomess.
After setting the link-layer encapsulation type of the pcap-geomess node, the traffic could be examined in Wireshark. The pcap-file of the hardcoded communication can be found in this repository under 'hardcoded_802_15_4.pcap'.
The first 2 packets aren't correctly formatted since that are ICMPv6 router sollication packets from picoTCP's Neighbour Discovery's DAD. When the time arrives that those 2 packets are correctly parsed, I'm heading in the right direction. ;-)

TODO: Dynamically encapsulate *to-send* data in an IEEE802.15.4-MAC frame.

##### 13 Aug 2015 20h -   Moved pico_dev_sixlowpan to picotcp/modules.

##### 14 Aug 2015 12h -   Simulated radio sends Association-Request.
Simulated Radio-driver now sends a association request when it's not the first node. First Node is identified bythe ID-command-line variable set to 0. This association request is send to the PAN-coördinator (0x000) to associate with a 802.15.4 PAN and retrieve a 16-bit short address given by the coördinator. In the afternoon, I will make the PAN coördinator respond to association requests and make all MAC framing a bit more accessible and generic. If this gets finished, I will probably be able to send my first IPv6 frames without compression/fragmentation/... over 802.15.4 PAN tonight.

##### 14 Aug 2015 15h -   Should I be responsible for starting/joining a PAN?

To set up a PAN, a coördinator must perform an Energy Detection scan, choose a proper channel, choose a PAN identifier and generate a 16-bit short-address for himself (0x0000). Then, a coördinator has properly set up a 802.15.4 PAN.

This isn't really a hard thing to do, but nevertheless application-dependent. An application developer can for example choose to use either beacon-enabled mode or beacon-disabled mode. To include this functionality in picoTCP would be too demanding and I think a right choise is to leave this starting of a PAN up to the application-developer.

To join a PAN, a 802.15.4-device must perform an Active or Passive Channel scan, send out an association request and wait for the PAN-coördinator to answer with an Association-response in order to only retrieve a 16-bit short address. Since every 802.15.4 should normally already have a unique 64-bit identifier, isn't the 16-bit short address a bit superfluous. Does the hard work of obtaining a 16-bit short address weigh up against disadvantage of having a somewhat bigger unique 64-bit address?

So I choose to not integrate this possibility of the association procedure in my implementation of 6LoWPAN. I think the 6LoWPAN should be mere an adaption layer and not really a 'maintaining-a-802.15.4-PAN'-layer.

So in my implementation of 6LoWPAN I will assume the application-developer himself has already set-up the PAN, either hardcoded (by setting the channel, PAN ID and not using a 16-bit short address by code) or dynamically (by performing an Active or Passive Channel scan and performing the association procedure).

So this means the 6LoWPAN adaption layer already assumes a couple of things:

1. That the radio has 64-bit link layer address, which is retrieved by '*getEUI64()*'.
2. That the radio already has configured the PAN identifier, which is retrieved by '*getPAN_ID()*'.
3. That the radio can already communicate in the PAN by conforming to the 2 previous assumptions.
4. That the radio possibly has a 16-bit short link layer address, which is retrieved by '*getSHORT16()*', and that if the radio hasn't got a 16-bit short this function will get the value 0xFFFF.
5. That the radio has a possibility to set the 16-bit short afterwards, which is set by '*setSHORT16()*', by means of 6LoWPAN Neighbour Discovery.

##### 14 Aug 2015 15h -   Removed simulated association procedure.
Already removed the simulated 802.15.4 Association Procedure, since I will configure the 802.15.4-devices by command line variables.

##### 17 Aug 2015 17h -   Sent my first 6LoWPAN packet.
Sent my first 6LowPAN packet over Geomess today. It's a Neighbour Solicitation ICMPv6 packet coming from picoTCP's Duplicate Address Detection (DAD). The packet is dynamically formatted and derives it's Link-Layer framing options an addresses from the IPv6 packet header.

##### 17 Aug 2015 17h -   Finished key issue *Simple IPv6 over 802.15.4*.
With the previous jrnl-entry about sending my first 6LoWPAN packet, i'm halfway completing key issue '*Simple IPv6 over 802.15.4*' as described in entry [11 Aug 2015 18h](https://github.com/jelledevleeschouwer/sixlowpan_log#11-aug-2015-18h-----6lowpan-development-process). The decapsulation procedure should still be implemented on the other side of the network.

##### 17 Aug 2015 22h -   Working on unframing of 802.15.4 packets [WIP].
Will be done tommorrow morning. Then, I can start on compression. Probably start off with compression scheme LOWPAN_HC1 and HC_UDP.

##### 19 Aug 2015 23h -   Architecture overhaul, use of PICO_FRAME.
Cleaned up my code a lot, and did some architectural changes. From now on, I will work with pico_frame as well just like ethernet. There's some minor twists though.

First of all, I translate the pico_frame to a 6LoWPAN-adaption layer frame *sixlowpan_frame*.

I have also created a seperate pico_device_init method just for 6LoWPAN-devices namely, *pico_sixlowpan_init*. This is because with the generic pico_device-init method I couldn't generate a Link-Local address from the EUI-64 address or from the short 16-bit address. With *pico_sixlowpan_init* this capability is there.

Then, I've added a *pico_sixlowpan_addr ** to the pico_device-structure. This to allow pico_device to have an extended 64-bit address or a short 16-bit address. This could've been in the pico_dev_sixlowpan itself, but putting it in pico_device adds the capability to detect whether or not the device is 6LoWPAN-device. Just like with the *eth*-member of pico_device.

When a frame needs to be sent to the device in '*devloop_sendto_dev*', the device is checked for a sixlowpan-address. When it has, the frame is sent to the 6LoWPAN adaption layer. This is just like when the device has an ethernet-address. But instead of sending the frame to *pico_stack*, the frame is sent directly to the 6LoWPAN-adaption layer.

Now in the sixlowpan_frame, I have a couple of buffers that allow me to work efficiently when inserting and removing chunks of the frames.

I'm still a bit looking for the best way to add 6LoWPAN headers and compressing IPv6 and next-header fields. But compressing should be done by the end of the week.

##### 21 Aug 2015 22h -   [WIP] LOWPAN_IPHC stateless compressing almost finished.
Stateless LOWPAN_IPHC compression is almost finished. Tommorrow-morning it will probably be done, there are still a couple of minor flaws and ugly code but I will fix that tommorrow. Then I can move on to LOWPAN_NHC in the afternoon and probably start with decoding LOWPAN_IPHC and LOWPAN_NHC somewhere around 16.00h. On Sunday I will write unit tests, and then on Monday I will finish the decompression schemes.

Like I said, now only stateless IPHC is implemented when I move on to 6LoWPAN-ND in september, I will make IPHC use Statefull Compression as well.

The code is starting to look pretty cool and I'm pretty satisfied about the structure and complexity, but it is getting big though. I think it would be much clearer if I split up IPHC, 6LoWPAN and IEEE802.15.4. But still, when I look into the code of other 6LoWPAN implementation they have A LOT more codebase :-)
