# [LightBlue Bean](http://punchthrough.com/bean/) Serial Message Protocol

* Documentation version: 0.3.2
* Created: 2014-06-25
* Updated: 2015-01-06
* Copyright: [Punch Through Design](http://punchthrough.com)
* License: [The MIT License](http://opensource.org/licenses/MIT)

# Overview

This document describes the protocol used to communicate with the Bean via its Bluetooth Low Energy serial characteristic.

# Attributes

Attribute                     | UUID
------------------------------|--------------------------------------
Serial Service                | A495-FF10-C5B1-4B44-B512-1370F02D74DE
Bean Transport Characteristic | A495-FF11-C5B1-4B44-B512-1370F02D74DE

The serial service holds one characteristic named "Bean Transport."  The Bean Transport characteristic properties are normally set to "notify" by the client to speed up data transmission.  The client writes to the transport characteristic using the write without response property.  

# App Message

The App Message is the highest level of abstraction. Arduino sketches written for the Atmel chip and programs written with the Bean SDK send and receive App Messages.

An App Message has the following components:

2 bytes    | 0-64 bytes
-----------|--------
Message ID | Payload

## Message ID
* 2 bytes
* Represents a specific predefined message type
* Read the [App Message Types](app_message_types.md) doc to learn more about App Message ID definitions

## Payload
* 0 to 64 bytes
* Holds the data payload sent to/from the Bean

# GATT Serial Transport (GST)

The GST is one level closer to the raw Bluetooth Low Energy characteristic. The GST layer places a header and footer around the App Message to allow message integrity to be verified on receipt.

A GST packet has the following components:

  1 byte | 1 byte   | 2-66 bytes  | 2 bytes
---------|----------|-------------|-----
  Length | Reserved | App Message | CRC

## Length
* 1 byte
* Represents the total length of the GST packet

## Reserved
* 1 byte
* Reserved for future use
* Default value: `0x00`

## App Message
* 2 to 66 bytes
* Full body of an App Message (message ID + data)

## CRC
* 2 bytes
* The 16-bit CRC of the preceding 4 to 68 bytes

# GATT Transport (GT)

The GT is the lowest level of abstraction in the Bean serial message protocol. GT packets are read and written via the Serial characteristic available on the Bean.

Although our GST packets can have lengths of up to 70 bytes, the Bluetooth Low Energy specification allows writes to have maximum lengths of 20 bytes. The GT layer handles this by splitting GST packets into 0 to 19 byte chunks and packaging them with a 1 byte GT Header, then sending the GT packet(s) sequentially.

A GT packet has the following components:

 1 byte     | 0-19 bytes
------------|---------
  GT Header | Payload

## GT Header
* 1 byte
* Contains metadata describing the context of the attached payload

## Payload
* 0 to 19 bytes
* Contains all or part of one GST packet

# GT Header

To guarantee data integrity, the GT layer needs to include some metadata to ensure that all packets are received and processed in the correct order. This metadata is contained within the 1-byte GT header.

A GT Header has the following components:

 1 bit | 2 bits        | 5 bits        
-------|---------------|--------------
 Start | Message Count | Packet Count

## Start
* 1 bit
* Set to `1` for the first packet of each App Message, `0` for every other packet

## Message Count
* 2 bits
* Increments and rolls over on each new GT Message (0, 1, 2, 3, 0, ...)
* Helps ensure packets are received in the right order

## Packet Count
* 5 bits
* Represents the number of packets remaining in the GST message
* Decrements to `0`, where `0` represents the final packet in the GST message
