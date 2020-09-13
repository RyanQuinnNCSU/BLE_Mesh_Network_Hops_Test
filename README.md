# BLE_Mesh_Network_Hops_Test
> Creating Self-Provisioned BLE Mesh firmware to test BLE Mesh network anlysis. 


##  Why Conduct Network Testing
BLE Mesh has lost of parameters (ttl, retransmit count, ect) that can effect the reliablily of network traffic. Too much BLE Mesh traffic will cause broadcast storms, while low retrasmit settings can cause some message to be missed. The Silabs test api's allow developers to experiment with different parameter settings and there affect on network performance. The hearbeat test demonstrated here shows the max and min number of hops it takes for a message to be sent between to nodes in a network.

## What is a Heartbeat Test?
In this context a heartbeat is a BLE Mesh test message. Using test APIs, when a heartbeat as been recieved by a device configured as a subscriber it will record the number of network hops between the publish node and the subscriber as well as the number of hearbeats recieved. The node that publish the heartbeat also has to be configured with key settings such as the number of heartbeats to publish, the time between publishing heartbeats and the destination address. The hearbeat test is useful in knowing what value to see TTL in your network and to test for network reliablity (did any messages get lost).


```
Heartbeat Example:

Hearbeat Message 1 reaches the subscriber in 2 hops.

					   Hop 1                                Hop 2
*****************                    *****************                    ***************
*Publishing Node*  --------------->  *In Between Node*  --------------->  *Subscrbe Node*
*****************                    *****************                    ***************

Hearbeat Message 2 reaches the subscriber in 1 hop.

*****************                    *****************                    ***************
*Publishing Node*                    *In Between Node*                    *Subscrbe Node*
*****************                    *****************                    ***************
      |																			  ^
	  |                                   Hop 1                                   |
	  -----------------------------------------------------------------------------
						

```

## Initial Setup
For network testing, it is often easier to self-provision your devices via the firmware. By setting network keys/settings in the firmware, the device are ready to be tested upon bootup. For an indept review of how to set up self-provisioned BLE Mesh firmware checkout my other repo: https://github.com/RyanQuinnNCSU/Self_Provision_BLE_Mesh_Device 

This project was built off on the linked repo and was devloped using Silabs' Simplicity Studio (IDE). The documentation provided here will only go over the new changes required to run the heartbeat test.

The hardware used for this project was Silabs Dev Kits that use the efr32mg12p432f1024gl125 chip.

<img src="/Photos/Dev_Kit_Photos/dev_kit.jpg" height="500">

## Heartbeat Test Requirments
The following code changes need to be impliented inorder to run a BLE Mesh network heartbeat test.

### Enable Mesh Test APIs

In order to use the heartbeat test APIs (or run devices as self-provisioned) you will need to enable the test APIs with  ```gecko_bgapi_class_mesh_test_init();``` 
Add this with the other API intit functions contained in ```gecko_bgapi_classes_init()``` (see soc-btmesh-sensor-client/app.c).


### Create Unique Unicast Addresses
The heartbeat source and destination address for most test should be unicast address of device in your test network. On self-provsioned firmware the unicast needs to be defined on bootup. 
To give each device a unique unicast address while only flashing one firware image the unicast should be derived from the Mac address of each node.
The Mac address of each device can be referenced using ```SYSTEM_GetUnique()```. One thing to note, the most significant bit of the unicast address must be 0. The solution is to only use the last 15 (not 16) of the MAC for your unicast. For example ```Unicast = (uint16_t)(SYSTEM_GetUnique() & 0x7FFF);``` (soc-btmesh-sensor-client/main.c).

One final note. ```#include "em_system.h"``` must be in added for ```SYSTEM_GetUnique``` to work. 

### Setup Hearbeat Publisher
The API call used to start publishing heartbeat messages is ```gecko_cmd_mesh_test_set_local_heartbeat_publication(publication_address, pub_count, pub_period, HB_TTL, features, netkey_index)``` (see soc-btmesh-sensor-client/app.c). Once this function gets called on the publishing node's hearbeat messages will be sent though out the network.

As the API suggest there are lots of parameters that control heartbeat publication. The most critical ones are summarized below:
* publication_address: Destination node (unicast) address. 
* pub_count: The number of heartbeats to send. Value of pub_count will result in 2^(x-1) messages getting published. Example, if pub_count = 5, then 2^(5-1) = 16 messages.
* pub_period: The time between publishing heartbeat messages. Value of pub_period will result in 2^(x-1) seconds between messages. Example, if pub_period = 2, then 2^(2-1) = 2 seconds between messages.
* HB_TTL: The time to live of the hearbeat messages (how many hops before packet stops getting retransmitted). 

Each of these parameters can be changed by writting to a GATT serivice on the publishing node (details provided farther down in the readme). While all these parameters have default values, atleast the publication_address will need to be written to before the test started.

In this repo the ```gecko_cmd_mesh_test_set_local_heartbeat_publication()``` gets called when button 0 on the silabs dev kit gets pressed.


### Setup Hearbeat Subscriber 
The API call used to start recieving heartbeat messages is ```gecko_cmd_mesh_test_set_local_heartbeat_subscription(subscription_source, subscription_destination, sub_period)``` (see soc-btmesh-sensor-client/app.c). Once this function gets called on the subscribing node will start listening for hearbeat messages with it's unicast address as the destination and record the number of hops it took for the message to arrive.


As the API suggest there are lots of parameters that control heartbeat subscription. 
* subscription_source:  The publishing node's unicast address. 
* subscription_destination: The subscribing node's unicast address.
* sub_period: How long the subscribing node will listen for heartbeats in seconds. Value of sub_period will result in the node listen for heartbeats for 2^(x-1) seconds. Example, if sub_count = 5, then 2^(5-1) = 16 seconds.

Each of these parameters can be changed by writting to a GATT service on the subscribing node (details provided farther down in the readme). While all these parameters have default values, atleast the  subscription_source and subscription_destination will need to be written to before the test started.

In this repo the ```gecko_cmd_mesh_test_set_local_heartbeat_subscription()``` gets called when button 1 on the silabs dev kit gets pressed.


### Configuring Heartbeat Publisher and Subscriber Settings via GATT Connect.

Key parameters for both the publishing and subscribing heartbeat APIs can and should be edited via GATT services. In this tutorial it is assumed that devlopers are fimiliare with what GATT services and characteristics are and how to use them. Typically a mobile app is required use GATT services. The app I typically use is "EFR Connect" developed by Silabs.

In this demo repo, 2 GATT services have been defined.

Hearbeat Publish: (Allows users to edit hearbeat publisher parameters
* UUID: bf93eaea-db28-4e72-8f0f-0aa8997a6720
* Service Characteristics: (all have read and write privilages)
	+ pub_addr
		- UUID: 81547b88-7cc6-4a57-a386-1bf21a70ae20
	+ pub_count
		- UUID: 9b2da3fc-257d-417e-b449-ca18950ed081
	+ pub_period
		- UUID: 418c7218-fae1-4f71-baae-a7ae55ee573a
	+ HB_TTL
		- UUID: c60b50a3-0313-44a9-b2fb-df065ecd1136


Hearbeat Subscribe: (Allows users to edit hearbeat subscriber parameters
* UUID: f38aa561-2acb-44d8-8e07-247d139d1be3
* Service Characteristics: (all have read and write privilages)
	+ subscription_source
		- UUID: 95c25690-cc30-4de3-80b1-6029acfc75a7
	+ subscription_destination
		- UUID: 1e9ea2d6-f740-4761-b8ba-b5ae97e17d24
	+ sub_period
		- UUID: 87e6d9d2-06e2-4f75-95a5-bf869dd576dd



Each parameters that has been assigned to a GATT service/charactersitic has a read and write handle. Below is the example code for handling reads and writes to pub_addr.

Writing to pub_addr:
*User on mobile app sends new value of pub_addr to publishing node using the GATT service.
*gecko_evt_gatt_server_user_write_request_id event triggered.
*if char that triggered event == pub_addr char then check the lenght of the write bytes
*pub_addr is an uint16_t (2 bytes). If the written data is not the size of the parameter then skip write request. Otherwise change value of pub_addr to written value.

```
case gecko_evt_gatt_server_user_write_request_id:
    	printf("GATT write request received \r\n");
    	int write_len = 0;
		...
		...
		...
		//Publish Address
		  if (gattdb_Pub_Addr == pEvt->data.evt_gatt_server_user_write_request.characteristic) {
			  write_len = (int)pEvt->data.evt_gatt_server_user_write_request.value.len;
			  if(write_len != 2){
				  DI_Print("   GATT WRT Fail    ",DEBUG_MESSAGE_ROW);
				  gecko_cmd_hardware_set_soft_timer(10*ONE_SECOND,TIMER_ID_CLEAR_DI_MESSAGE,1);
				  gecko_cmd_gatt_server_send_user_write_response(pEvt->data.evt_gatt_server_user_write_request.connection, pEvt->data.evt_gatt_server_user_write_request.characteristic, bg_err_invalid_param);

			  }
			  else{
				  publication_address = (pEvt->data.evt_gatt_server_user_write_request.value.data[1]) | (pEvt->data.evt_gatt_server_user_write_request.value.data[0] << 8);
				  printf("publication_address written as %x \n\r", publication_address);
				  gecko_cmd_gatt_server_send_user_write_response(pEvt->data.evt_gatt_server_user_write_request.connection, pEvt->data.evt_gatt_server_user_write_request.characteristic, bg_err_success);
			  }

		  }
```



Reading pub_addr:
*User on mobile app request to read the value of pub_addr using the GATT service.
*gecko_evt_gatt_server_user_read_request_id event triggered.
*if char that triggered event == pub_addr char then send the value of pub_addr in read response. 



```

case gecko_evt_gatt_server_user_read_request_id:
    	printf("GATT read request received \r\n");
		...

		//Publish Address
		  if (gattdb_Pub_Addr == pEvt->data.evt_gatt_server_user_read_request.characteristic) {
			  gecko_cmd_gatt_server_send_user_read_response( pEvt->data.evt_gatt_server_user_read_request.connection, gattdb_Pub_Addr, 0, 2, &publication_address);
			  }
```


### Seeing the Code in Action
Lets run though and example Heartbeat test with a small BLE Mesh self-provisioned network of 3 Silabs dev kits.

1. Flash the firmware binary onto each device. Once the firmware has been flash to your devices, check to the serial log to make sure no errors have occured. Below is an example of a self-provisioned bootup with no errors.


<img src="/Photos/Putty_Logs/Unicast_550f_Bootup.PNG" height="500">

2. Set the Publisher:
With the network up and running, choose a node to publish the heartbeat messages and define the parameter values. This will require using the GATT service discribed in a previous section of the readme.

In this example there are 3 devices to choose from. Below is a scan taken from the "EFR Connect" mobile app showing all 3 devices and there unicast address in hex (note this scan shows a custom name beacon set up in the repo).

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-141708.png" height="500">

Lets choose the device with unicast 550f as the publisher and device 4440 as the subscriber.

To set the publisher settings press the "connect" button next to 550f.

After a brief loading screen you should see contents similare to the screenshot below listing the various GATT services of the 550f node.

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142532.png" height="500">

Lets scroll until we find the heartbeat publish GATT service (UUID bf93eaea-db28-4e72-8f0f-0aa8997a6720) *second from the bootom

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142546.png" height="500">

Tap on the publish service to reveal the different publisher parameter characterstics. 

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142555.png" height="500">

The mobile app doesn't display names for these charactersitics only the UUID. Please reference the earlier sections of this document to match publish paramters with the char UUID.

The only publish parameter that can't use the default value is pub_addr. Its char UUID is 81547b88-7cc6-4a57-a386-1bf21a70ae20. Press on that char to write the destination address 0x4440.

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142617.png" height="500">

After pressing send we should see on the serial log that the value of pub_address on 550f has been updated.
 
<img src="/Photos/Putty_Logs/550f_write_pubaddr.PNG" height="500">

While we can alter the other publish parameters for this example they will be kept to there default values. lets disconnect the GATT connect from the 550f node and the mobile app. Press the back button to return to the services window and press "disconnect".

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142742.png" height="500">

3. Set the Subscriber:

In this example, the device with unicast 4440 will be the subscribing node. To set the subscribing settings press the "connect" button next to 4440.

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-141708.png" height="500">

Once the GATT connection has been established, scoll to the subscription service (UUID f38aa561-2acb-44d8-8e07-247d139d1be3)

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142836.png" height="500">

Now set the parameter values of subscription_source (UUID 95c25690-cc30-4de3-80b1-6029acfc75a7) and subscription_destination (UUID 1e9ea2d6-f740-4761-b8ba-b5ae97e17d24)

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142853.png" height="500">

<img src="/Photos/Blue_Gecko_Screenshots/Screenshot_20200912-142926.png" height="500">

After pressing send we should see on the serial log that the value of pub_address on 550f has been updated.
 
<img src="/Photos/Putty_Logs/4440_sub_write.PNG" height="500">

The sub_period can also be changed, but for this test we will use the default value. 

4. Run the Test:

Now that we finally have the publish and scubscriber nodes configured we can start the test!

As stated in a previous section presses button 0 on a dev kit will run the publish API call and button 1 will run the subscribe API. 

Button 1 is pressed on device 4440 and it starts to listen for hearbeat messages for the default period of time.

Then button 0 is pressed on device 550f and it starts to publish the default number of hearbeat mesages. 

The LCD on the dev kits shows debug messages saying that 550f is publish heartbeats and 4440 is receiving hearbeats.

<img src="/Photos/Dev_Kit_Photos/HB_prints.jpg" height="500">

Once the sub_period has run out (by default 16 seconds) the gecko_evt_mesh_test_local_heartbeat_subscription_complete_id event is tirggerd in node 4440. From this event it will print the min/max hops and the number of messages recieved. 

<img src="/Photos/Putty_Logs/4440_test1.PNG" height="500">

Notice that in this test the min and max number of hops was the same at just one hop. This means that 3rd device (unicast 562a) in the network was never used to forward packets from 550f to 4440. This is becasue I had the publishing and subscribing device write next to eachother.

In another test I conected, I spread out all 3 network devices and changed some additiona parameters via the GATT chars. As you can see below, now the max number of hops is set to 2 (meaning 550f -> 562a -> 4440).

<img src="/Photos/Putty_Logs/4440_test2.PNG" height="500">

# Wrap Up
Hope this helps, best of luck in your network testing!

# Contact Info
Email: rtquinn2@ncsu.edu
