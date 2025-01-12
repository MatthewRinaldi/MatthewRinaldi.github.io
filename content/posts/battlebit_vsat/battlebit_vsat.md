---
title: Making a Battlebit Remastered Minimap with Packet Sniffing
date: 2024-06-12T20:03:12-05:00
---

*This post is for educational purposes only.*

A few months ago I purchased Battlebit Remastered, which is a "massive multiplayer FPS." The game is has provided me with a lot of enjoyment, but I have one gripe: there is no minimap. Pressing the 'M' key does provide a map but it is not exactly mini: it fills up half your screen and makes it virtually impossible to play while activated. I thought it would be a fun project to try and make my own fully passive minimap: no hooks or inspecting process memory. Just pure packet sniffing. I also challenged myself to not disassemble and reverse the game's binary, which is why all my findings are closer to educated guesses rather than absolute truths.


{{<figure 
	src="https://static.wikia.nocookie.net/callofduty/images/5/52/Satellite_CoDO.png"
	caption="My inspiration was the Orbital VSAT killstreak from Black Ops 2, which was an upgraded UAV which displayed the direction all the players were facing."
>}}

---

## Reversing the Packet Format

The first step was to see what kind of traffic the game generated while playing in a multiplayer server. I used Wireshark to capture and analyze the packets. The game data was sent over UDP and was completely unencrypted, which made my life a whole lot easier.


I discovered that each UDP packet contains chunks of data in a TLV-esque format (more like LTV). There were several different kinds of packet types, but one type stood out in particular. Below is an example of a packet that contains players' location:
```
0000   02 00 00 29 00 0b 38 01 00 12 28 b6 c3 49 47 69   ...)..8...(..IGi
0010   42 cc e6 da c3 ff 7f ff 7f ff 7f 03 f5 4e 7c 40   B............N|@
0020   4c 3e 50 7d 7f 7f 7f 7f 00 14 03 00 00 ff 29 00   L>P}..........).
0030   0b 09 04 00 80 4f 00 c4 6f 3d 7b 42 99 b6 0b c4   .....O..o={B....
0040   ff 7f ff 7f ff 7f d9 fc a5 7a b0 9b 1d 37 7b 7f   .........z...7{.
0050   7f 7f 7f 02 06 01 00 00 ff 0e 00 14 e4 0c 49 58   ..............IX
0060   5d 8d fd 3e 3c 85 e5 00 00 0c 00 14 50 14 b2 55   ]..><.......P..U
0070   ae 8d 62 3e 03 68 12                              ..b>.h.
```

Figuring out the format of these packets took a bit of work and intuition. I found the coordinate data quite easily: there are three adjacent floats in the chunk, corresponding to x, y, and z coordinates. To get the full "VSAT" effect, I needed the heading of each player too, which was a bit trickier to find. I tediously stared at the hex digits of the chunks while facing my character in different directions, attempting to recognize any sort of pattern. Eventually, I figured out that the heading was encoded as a fractional 16-bit integer (i.e. number / 0xffff). The annotated packet below shows some of the fields I was able to decode. The player location packets sent from the client were in a similar format with some very minor differences.

{{<figure 
	src="/mmfps_uav/packet_breakdown.png"
	caption="Packet with a few of the fields decoded. I didn't reverse the binary, so these are just guesses."
>}}

To process the packets in real time, I made a Python script that utilized the scapy library's `sniff()` function. Combined with some basic struct `unpack()`'s, I decoded all the data I needed to generate the minimap. I ran into an issue of my computer freezing after sniffing for an extended amount of time and after some googling, [found out why](https://stackoverflow.com/a/31263464).

```py
# without store=0, sniff will gradually gobble up all memory :/
sniff(iface="enp0s31f6", filter="udp", prn=decode_incoming, store=0)
```

## Rendering

With all the coordinate data extracted from the network, the next step was to display the data in a minimap-like format. I had initially planned to use the curses library, but ended up switching to WebGL, which I was a bit more familiar with. I used the [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) to transfer the data from Python to JavaScript. The WebGL code is straightforward, it rerenders new positions each time it receives a WebSocket messages.

I built a simple user interface to control some parameters of how the minimap was rendered. I could control:
- zoom level
- centering the minimap on the player vs the center of the map
- toggling heading (triangles vs dots)

{{<figure 
	src="/mmfps_uav/site.png"
	caption="The full site. Amazing UI by yours truly."
>}}

Below are some clips of the minimap in-game. The first is the full map view, and second is focused on my characters position.

<video width="500" height="500" controls muted style="display: block; margin-left: auto; margin-right: auto; width: 50%;">
	<source src="/mmfps_uav/spawning_in.mp4" type="video/mp4">
	Your browser does not support the video tag.
</video>

You can see the teams moving from their spawns to different parts of the map and my character spawning in the last few seconds.

<video width="500" height="500" controls muted style="display: block; margin-left: auto; margin-right: auto; width: 50%;">
	<source src="/mmfps_uav/in_game.mp4" type="video/mp4">
	Your browser does not support the video tag.
</video>

I had to relearn some high school trigonometry for the player centred map.

There are a few *minor* usability issues: 
- the triangles' movement is not very smooth
- there is no background showing buildings/obstacles on the map, which makes it difficult to judge distance
- the minimap needs to be flushed often because there is an issue with stale packets being in the queue
- there is a 'shadow' following my character's position
- and last but not least, I could not figure out was how to determine the corresponding team of a position packet, therefore all packets that come from the server (including my teammates) are shown in red.

I am sure that with additional time and effort, I could sort all these issues out. But I was planning on making a proof of concept, and the concept was proofed enough for me.

---

## Conclusion

To actually use this, I had to forward the packets from my gaming PC to my laptop to offload the packet processing, otherwise my framerate would grind to a halt. It wasn't the prettiest solution but I could make quick glances at my laptop screen and see what was going on.

It turns out you can still go pretty far even in this day and age just sniffing packets. I believe this project is *technically* not breaking the EULA, but I would never try this on actual servers. As I said, educational purposes and what not.
