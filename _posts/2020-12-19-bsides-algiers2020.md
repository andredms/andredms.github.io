---
layout: post
title: BSides Algiers 2020 - Forensics 
description: BSides Algiers came to a close earlier, and although having slightly harder challenges than what I’ve done before, it was a great learning experience. This writeup will be focusing on the forensics category and the challenge 'It’s Complicated My Pal'.
image: https://i.imgur.com/iQSJG9W.png
---

BSides Algiers came to a close earlier, and although having slightly harder challenges than what I’ve done before, it was a great learning experience. This writeup will be focusing on the forensics category and the challenge 'It’s Complicated My Pal'.

My team for this CTF was 0xTBD. @Pix and I worked on this particular solution together which involved network capture analysis, extracting an image sent via ICMP packets and cracking a .zip file password. 

# The Challenge
**Name:** It’s Complicated My Pal

**Category:** Forensics


**Points:** 500


**Requirements:** Python, [Wireshark](https://www.wireshark.org/download.html), [Scapy](https://scapy.readthedocs.io/en/latest/installation.html), [fcrackzip](https://github.com/hyc/fcrackzip).


>Hey pal, I was sniffing some packets and stumbled upon some [weird traffic](https://drive.google.com/file/d/1nRXcYuAYLQUsgEc9wC_wfP0BYWfv_azS/view?usp=sharing), no idea what it is though. Could you take a look at it? It's complicated for me and I can't analyze it by myself. **Author:** [hfz](https://hfz1337.github.io/about)

# Initial Analysis
The link in the challenge description contains capture.pcap, which we will use Wireshark to analyse. One thing I like to do as soon as I get network traffic is go into Analyse -> Conversations and sort by port number to get a better idea of the type of traffic we’re dealing with. 

![image](https://i.imgur.com/DqbEKUV.png)

Immediately we can see there’s lots of HTTPS traffic - which we can ignore as without a key to crack it, it’s effectively useless. The following filter works via dismissing all traffic going to port 443 (HTTPS). 

```
!tcp.port==443
```

This greatly streamlines the amount of packets we have to look at and uncovers some odd behaviour. We’re greeted by 25,000-ish ICMP packets, which if we turn the filter off, look as if they’ve been covered up by HTTPS traffic. Perhaps the attacker was trying to disguise their mischief?

![image](https://i.imgur.com/APYBiZJ.png)
**No filter**

![image](https://i.imgur.com/iQSJG9W.png)
**Filter applied**


Outside of a CTF environment, this may have looked like some form of DDoS attempt ([ping flood](https://www.imperva.com/learn/ddos/ping-icmp-flood/)), however upon looking at the first ping we can see 'flag.jpg' in the data section of the packet. 

![image](https://i.imgur.com/o8zjNlT.png)

This lets us know that there’s some form of exfiltration going on, or [ICMP tunnelling](https://en.wikipedia.org/wiki/ICMP_tunnel), as a ping request wouldn’t typically have a crafted payload like the one above and there's a ridiculous amount of them. It’s also worth noting we stumbled upon ``flag.jpg`` by complete chance.

One thing we immediately tried looking for is the .jpg header signature ('ffd8') to check that it wasn’t a red herring. This can be done via ctrl (or cmd) + f, typing in 'ffd8' and setting the filter to 'Hex Value' (as per the screenshot below). Packet 1908 contained it. 

![image](https://i.imgur.com/xVNMYs1.png)

# Extracting Data
We now knew that there was an image being tunnelled through ICMP, but as you could imagine, there was the issue of actually extracting it from the network capture. This is where we came up with some great, and not so great ideas.

Firstly, I figured that all the data we needed would be being transferred via echo requests from 192.168.1.200 -> 185.245.99.2. We wouldn’t really need to worry about the replies from the client or the (no response found!) packets, as they wouldn’t actually contain the data being transferred. I created a new refined filter so we could prepare the ICMP payloads for dumping:

```
icmp && ip.src != 185.245.99.2 && !icmp.resp_not_found
```

The ``icmp`` part gets us ICMP traffic exclusively, ``ip.src != 185.245.99.2`` ensures that we don’t get any echo replies from 185.245.99.2 (who 192.168.1.200 is sending the data to) and ``!icmp.resp_not_found`` filters out all of the (no response found!) packets. We now had data ripe for the dumping. 

Prior to this challenge, I had never extracted payloads out of packets, but Googling how to do this gave the option of File -> Export Packet Dissections -> As Plain Text. I unchecked all but the following options:

![image](https://i.imgur.com/6deWChv.png)

…which then dumped the hex payloads from the packets to dump.txt - below is a small sample of what the file looked like:

![image](https://i.imgur.com/Um4uudc.png)

I knew that in order to get the data we wanted, we needed to cut a few things out:

1) The 0000, 0010, 0020 ect. part to the left hand side of the hex.


2) The ASCII decoded part to the right hand side of the hex.


3) All the ICMP header fluff that stood between us and the flag.

After much messing around I came up with the following script using ``awk``, that did exactly the above, except for the last:
```
awk '{x = ""; x = substr($0, 5, 50); gsub(/ +/, "", x); print x}' dump.txt > trimmed.txt
```

Breaking this down:

```
x = substr($0, 5, 50); 
```

Creates a substring from characters 5 - 50. We start at 5 because we don’t require 0000 + the space (five total characters), and then go all the way to 50 as there’s 32 hex characters, 16 spaces and two spaces between that and the ASCII decoded string. 

```
gsub(/ +/, "", x);
```

Substitutes all spaces for NULL characters in x.

```
print x
```

Prints the final string, which we then redirect the output of to trimmed.txt (via '> trimmed.txt'). 

After running this, we now get a trimmed text file which looks a little something like…

![image](https://i.imgur.com/swWCARk.png)

A simple tr command can get rid of the newlines for us: 

```
tr -d "\n" < trimmed.txt > trimmed2.txt
```

(-d for delete, "\n" for new line)

![image](https://i.imgur.com/uEvVXs1.png)

We were now one step closer to getting the flag, however the ICMP header fields were still a problem (i.e. the dead0000beefcafe0000babe IPv6 addresses). Looking at the structure of an ICMP packet, we saw that we could ignore the first 84 characters as they were irrelevant and get the remaining 96 characters (e.g. the blue highlighted part in the non-modified payload in Wireshark), which would give us the data we needed:

![image](https://i.imgur.com/IUDRT1C.png)

# More Automation 
Before continuing down this painful path of writing scripts to edit .txt files, Pix suggested that we use Scapy to grab the pcap data and dump the hex we actually want that way (a far more elegant and logical solution). I’m not entirely sure why I didn’t think of this myself but I'll definitely be using this approach in the future...we came up with the following Python script:

```python
from scapy.all import *
import base64

capture = rdpcap('capture.pcap') # our pcap file

output = open('output.bin','wb') # save dumped data as bytes to output.bin

for packet in capture:
    # if packet is ICMP and type is 8 (echo request)
    if packet.haslayer(ICMP) and str(packet.getlayer(ICMP).type) == '8': 
        output.write(packet.load) # write packet data to output.bin
 ```

Upon running the script, which essentially did what I was doing before but much quicker, we finally got the data we needed in the form of output.bin. To ensure we’d saved it with the right extension we ran:

```
file output.bin
```

which output:

```
output.bin: Zip archive data, at least v2.0 to extract
```

Turns out, it wasn’t actually a .jpg we were getting - but rather a .zip file which contained a .jpg. We thought that surely after renaming it to ``output.zip`` we’d be able to extract the file and grab the flag no problem, however it wasn’t so easy as the file was password protected.

# Cracking Passwords
We weren’t about to dive back into Wireshark to try find a key for the .zip somewhere, as it wasn’t even guaranteed to be there, so I did some Googling for a .zip cracker tool and came across the following: [frackzip](https://github.com/hyc/fcrackzip).

As the name suggests…it cracks .zip files. I decided it’d be best to run it against a common list of passwords, so I used [rockyou.txt](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt) and ran the program with the following flags:

```
fcrackzip -v -u -D -p rockyou.txt output.zip 
```

(-v for verbose, -u to unzip and weed out wrong passwords, -D for using a dictionary and -p to provide a password file).

Three minutes later, we got a match with 'craccer':

```
found file 'flag.jpg', (size cp/uc 225389/225940, flags 9, chk ac92)
checking pw buday1                                  

PASSWORD FOUND!!!!: pw == craccer
```

We unzipped the archive, opened the image and got the flag!

![image](https://i.imgur.com/ztY0ANV.jpg)

# What did I learn?
1) There's always a simpler or more automated way to filter network traffic (I've been brushing up on Scapy since completing the challenge). 


2) ICMP tunelling is super cool.


3) Googling gets the job done. 


4) Useful command line tools such as ``awk`` and ``tr``, as well as ``xdd`` (after the challenge was over). 
