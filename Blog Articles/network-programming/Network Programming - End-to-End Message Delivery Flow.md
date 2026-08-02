# Network Programming

In this illustration we will see how an email process (email app/program or browser tab) from node A communicates to email process at node B. Basically what happens when person using node A types "hi" message and sends to person using node B over different separated isolated locations.

Person (Node A):

message = "hi"

> clicks send button in email program UI

## Email process (Node A):

Serializes the python `Message` object to JSON/ protocol buffers given the email program is build on python.

This happens in Application layer which includes (View layer -> Service Layer).
						  ---------------------------
							       |
							       |
							       v
							

## Byte Encoding before streaming to sockets over TCP with IP

### 1. Char to ASCII conversion:

'h' ~ 104
'i' ~ 105

|      | 128  | 64   | 32   | 16   | 8    | 4    | 2    | 1    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 'h'  | 0    | 1	 | 1    | 0	   | 1	  | 0    | 0    | 0    |
| 'i'  | 0	  | 1    | 1    | 0	   | 1	  | 0    | 0    | 1    |


### 2. ASCII to Base2 conversion:

'h' ~ 104 -> 01101000
'i' ~ 105 -> 01101001


### 3. Base2 to bytes stream:

"hi" ------> 01101000 01101001


## TCP/IP socket transport:

Converted bytes will be streamed via socket over TCP with destination IP and port from source IP and port.

Which then later transports through Network Stack (physical wires) to destination IP:Port.

And then same process repeats but in reverse:

The email server process in Node C listens the byte stream since email process from Node A targets to this destination and then decodes and deserialises into destination  (Node C) process python objects.

## Node C process:

Once Node C (email server) receives the bytes stream and process (decodes/deserialise) the bytes it updates the database and then again convert (serialise/encode) the message from Node A and then push the  bytes to Node B active connection socket.

## Final message delivery:

Node B active socket connection receives the bytes pushed from Node B and then process (decodes/deserialise) the bytes and render message "hi" from person ~ Node A.