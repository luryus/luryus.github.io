+++
date = 2025-10-19
updated = 2025-10-19
title = "Loopback UDP broadcasts and multiple listeners"
authors = ["Lauri Koskela"]
draft = true

[taxonomies]
tags = [ "linux", "networking" ]
+++

I ran into a situation where multiple processes needed to receive all UDP datagrams sent from a service running on the same host. The sender already set `SO_BROADCAST`, and I thought setting the target host to `127.255.255.255` should be enough to make things work. And of course, the clients must use the same host and specify `SO_REUSEADDR` so that they receive the packets. But the reason _why_ this works is more interesting.

<!-- more -->

UDP broadcasts on the loopback interface are  OS-specific, and can't be relied upon to work on all platforms. I couldn't find any clearly written information about this on the Web, probably just because of this platform specificity. And most of the slop machines[^slop-machines] I tried did not generate anything useful when asking about UDP broadcasts on localhost. Various Stack Exchange posts indicate that [this works on Linux but not on macOS](https://serverfault.com/questions/553898/what-is-the-equivalent-of-127-255-255-255-for-os-x-machines-so-i-can-test-broadc).

The _correct_ way to handle this kind of situation would probably be multicast sockets, or even some other IPC solution. But UDP broadcasts was what I had here.

## How It Works

1. In the UDP sender, enable `SO_BROADCAST` on the UDP socket. Send the packets to the host `127.255.255.255`, and a port of your choosing.
2. In the receivers, enable `SO_REUSEADDR` on the listening socket. Bind the socket to `127.255.255.255` and the appropriate UDP port.

With this specific configuration, every active listener gets every datagram sent by the sender. Listeners can freely come and go, and everything just works.

If the sender and receivers use any other address in the `127.0.0.0/8` range, only one of the receivers will get the datagrams. So using the "broadcast" address (`127.255.255.255`) is important, even though this is not really broadcasting.

Here's a Python script that demonstrates this behaviour. Run the sender with `python3 udp.py sender`, and receivers with `python3 udp.py receiver`.
```py
import socket
import time
import argparse

PORT = 5005
MESSAGE = b'Hello from sender'
BROADCAST_IP = '127.255.255.255'

parser = argparse.ArgumentParser()
parser.add_argument('mode', choices=['sender', 'receiver'])
args = parser.parse_args()

if args.mode == 'sender':
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
    while True:
        s.sendto(MESSAGE, (BROADCAST_IP, PORT))
        print(time.time(), "Sent message")
        time.sleep(1)
else:
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((BROADCAST_IP, PORT))
    while True:
        data, addr = s.recvfrom(1024)
        print(time.time(), f"Received from {addr}: {data.decode()}")
```

My understanding is that broadcast traffic on the loopback interfaces is somewhat undefined behaviour. The loopback interface does not even have the `BROADCAST` flag like other interfaces do:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN mode DEFAULT group default qlen 1000
    link/ether 02:00:f0:9f:92:a9 brd ff:ff:ff:ff:ff:ff
    altname enx0200f09f92a9
```

It looks like the UDP stack in the kernel still runs the broadcast logic for loopback datagrams anyway. `127.255.255.255` is handled as the broadcast address for the loopback network, and therefore all listeners receive the datagrams.

## OS Compatibility

I tested older Linux versions and some other OSes to get an idea of how usable this thing is.

- **Modern Linux kernels** work (I tested on 6.16.x)
- **Older Linux versions** work. I tested Ubuntu 14.04 with kernel 3.13.0. So Linux has worked this way for a long time, and hopefully will keep working.

- **FreeBSD** does not seem to have this behaviour. [Here's an old mailing list post that touches upon this topic.](https://mail-archive.freebsd.org/cgi/getmsg.cgi?fetch=249789+0+archive/2002/freebsd-net/20021208.freebsd-net)
- **Windows** (11) has its own broadcast specialties. Unlike Linux, Windows forwards broadcast datagrams to sockets bound to a single host address[^windows-issue]. So broadcasts sent to `127.255.255.255` can be received by sockets bound to `127.0.0.1`. But sockets can't bind to `127.255.255.255`. So the local broadcasts work, but a bit different to Linux.
- **MacOS:** I don't have a Mac to test with, but [the Internet tells me](https://serverfault.com/questions/553898/what-is-the-equivalent-of-127-255-255-255-for-os-x-machines-so-i-can-test-broadc) macOS does not support the local broadcasts. I would imagine the behaviour is somewhat similar to FreeBSD.

Looks like there's no simple cross-platform solution for this. Better to just switch to multicasts if this kind of functionality is needed on OSes other than Linux or Windows.


[^slop-machines]: Claude was the winner here, pointing me directly to the correct solution. ChatGPT and Gemini just yapped about multicast, which is technically a way to achieve the same results, but was not what I asked. I'm sure I could get the right code out of them with more prompting, but the problem was that my initial question (before I knew if this was even possible) did not produce the right answers.

[^windows-issue]: [Dotnet Runtime issue #83525](https://github.com/dotnet/runtime/issues/83525#issuecomment-1487601420) erxplains this difference.
