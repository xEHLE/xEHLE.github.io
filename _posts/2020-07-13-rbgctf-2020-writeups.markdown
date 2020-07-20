---
layout: post
title: "rgbCTF 2020 Writeups"
categories: ctf
---

## Introduction {#intro}
{: style="text-align: center"}

| Challenge Name | Category | Solves | Points |
|:--------------:|:--------:|:------:|:------:|
|[Adventure](#adventure) | Misc | 21 | 495|
|[Picking Up The Pieces](#picking-up-the-pieces) | Misc | 93 | 403|
|[PI 1: Magic in the Air](#magic-in-the-air) | Forensics/OSINT | 52 | 470 |
|[PI 2: A Series of Tubes](#a-series-of-tubes) | Forensics/OSINT | 22 | 495 |
|[Robin's Reddit Password](#robins-reddit-password) | Forensics/OSINT | 30 | 490 |
|[Alien Transmission 1](#alien-transmission-1) | Forensics/OSINT | 158 | 219 |
|[Typeracer](#typeracer) | Web | 184 | 119 |

----------------

## Adventure {#adventure}
{: style="text-align: center"}
#### Category: Misc | Solves: 21 | Points: 495
{: style="text-align: center"}

----------------

{:refdef: style="text-align: center;"}
![Adventure challenge description](https://i.imgur.com/jHrTIPk.png){: .imgCenter}
{: refdef}

Right off the bat, there's a huge hint in the description with the weird capitalization that spells out ATARI.  Okay, so we can check to see if this is actually an Atari game by throwing it in [an emulator](https://stella-emu.github.io/downloads.html).  We pop it in and lo and behold, we have ourselves a game!

{:refdef: style="text-align: center;"}
![Breakfast game for Atari](https://i.imgur.com/PfStxS5.png){: .imgCenter}
{: refdef}

So what is immediately obvious is that this is not a standard game (I mean your character sprite is just the number 1.)  Any time there are customized/edited assets inside a game, I try to "work backwards" and think of what sort of tools are out there to do this specific thing.  After a bit of searching, I found [Hack-O-Matic III](https://www.romhacking.net/utilities/723/) on [Romhacking.net](https://www.romhacking.net). It's a simple program that shows the ROM literally bit-by-bit, making it easier to spot sprites.  Sure enough, that leads us to the flag. 

{:refdef: style="text-align: center;"}
![The flag](https://i.imgur.com/Q3n00SO.png){: .imgCenter}
{: refdef}

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------

## Picking Up The Pieces {#picking-up-the-pieces}
{: style="text-align: center"}
####  Category: Misc | Solves: 93 | Points: 403
{: style="text-align: center"}

----------------

{:refdef: style="text-align: center;"}
![Challenge description](https://i.imgur.com/Fj5d8oI.png){: .imgCenter}
{: refdef}

Boy oh boy, it's a pathfinding puzzle! Usually when someone wants to find the most efficient path between two points, they're going to want to use Dijikstra's Algorithm. It's the algorithm at the heart of Waze, Google Maps, and countless other applications, so you know it has to be good. 

If you want learn to more about it, [check out this great video by Computerphile](https://www.youtube.com/watch?v=GazC3A4OQTE).

{:refdef: style="text-align: center;"}
![Sample path data](https://i.imgur.com/qjGVq1c.png){: .imgCenter}
{: refdef}

We start off by opening `map.txt` to see what we're working with, and it appears to be fairly standard stuff outside of the strings at the end. The first two numbers are the intersections that are connected and the third number is the distance between the two. Luckily Dijkstra's algorithm is popular enough to have thousands of libraries/packages/code snippets to reference.  We still have to write code of course, but [RyanCarrier/dijkstra](https://github.com/RyanCarrier/dijkstra) does the real heavy lifting here.

{% highlight go %}
package main

import (
	"bufio"
	"fmt"
	"log"
	"os"
	"strconv"
	"strings"

	"github.com/RyanCarrier/dijkstra"
)

func main() {
	// initialize the map and 200,000 numbered points
	graph := dijkstra.NewGraph()

	for i := 1; i < 200001; i++ {
		graph.AddVertex(i)
	}

	// open file with path data
	file, _ := os.Open("map.txt")
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		x := scanner.Text()
		y := strings.Split(x, " ")
		// convert strings to int
		z1, _ := strconv.Atoi(y[0])
		z2, _ := strconv.Atoi(y[1])
		z3, _ := strconv.Atoi(y[2])
		// routes needs to be added both ways
		graph.AddArc(z1, z2, int64(z3))
		graph.AddArc(z2, z1, int64(z3))
	}

	// find and print the best path
	best, err := graph.Shortest(1, 200000)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(best.Path)

}

{% endhighlight %}

{:refdef: style="text-align: center;"}
![Sample path data](https://i.imgur.com/FDOl7vI.png){: .imgCenter}
{: refdef}

We run our code and it spits out the optimal path, just as we hoped. All that's left is to write a quick one-liner to spit out the relevant lines and we're done!

The flag is : `rgbCTF{1m_b4d_4t_sh0pping}`

{:refdef: style="text-align: center;"}
![Sample path data](https://i.imgur.com/WQiP9NQ.png){: .imgCenter}
{: refdef}

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------

## PI 1: Magic in the Air {#magic-in-the-air}
{: style="text-align: center"}
####  Category: Forensics/OSINT | Solves: 52 | Points: 470
{: style="text-align: center"}

----------------

{:refdef: style="text-align: center;"}
![Challenge text](https://i.imgur.com/6EMB1Ud.png){: .imgCenter}
{: refdef}

Right off the bat we know we're dealing with some sort of sniffed wireless traffic. Opening the unzipped file in a hex editor confirms that we're dealing with a Bluetooth capture, so we can go ahead and open it up in Wireshark and get a better idea of what exactly we're working with.

{:refdef: style="text-align: center;"}
![File in hex editor](https://i.imgur.com/OzIlchO.png){: .imgCenter}
{: refdef}

{:refdef: style="text-align: center;"}
![Bluetooth capture in Wireshark](https://i.imgur.com/SDHijwo.png){: .imgCenter}
{: refdef}

The source is named G613 and we know it's a Human Interface Device, a quick Google search will show that it's a [wireless keyboard](https://www.logitechg.com/en-us/products/gaming-keyboards/g613-wireless-mechanical-gaming-keyboard.html). A quick once-over of the file reveals that the most commonly received message is this "Rcvd Handle Value Notification" which has the same 12-byte header and a constantly changing 13th byte. This is easily confirmed by flipping through packets while referencing [this keymap](https://github.com/greatscottgadgets/libbtbb/blob/master/python/pcaptools/btaptap) from the [greatscottgadgets/libbtbb](https://github.com/greatscottgadgets/libbtbb/) repo on GitHub. We convert the file to hex and then write this snippet of code to regex out the information we need and compare it to the keymap dictionary we found earlier.

{:refdef: style="text-align: center;"}
![Python code to extract keystrokes from Bluetooth capture](https://i.imgur.com/G4W4aCA.png){: .imgCenter}
{: refdef}

The result provides us with the phone number and country it's from. A quick Google search reveals that +46 is the dialing code for Sweden, so we have everything we need to solve this challenge and start on the next!

{:refdef: style="text-align: center;"}
![Extracted keystrokes](https://i.imgur.com/r0pCHXC.png){: .imgCenter}
{: refdef}

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------

## PI 2: A Series of Tubes {#a-series-of-tubes}
{: style="text-align: center"}
####  Category: Forensics/OSINT | Solves: 22 | Points: 495
{: style="text-align: center"}

----------------

{:refdef: style="text-align: center;"}
![Challenge text](https://i.imgur.com/s1Kjle4.png){: .imgCenter}

The challenge before this had us extract a suspect's phone number from a Bluetooth capture file. A quick Google search doesn't return anything of value, but that's okay because there's a lot of places you can go with a phone number.  We put the number into our phone and open Snapchat, one of the million social media apps that want access to our contact list.

{:refdef: style="text-align: center;"}
![Added contact to phonebook](https://i.imgur.com/vV8ZD86.jpg){: .imgCenter}
{: refdef}

Lo and behold, good ol' Donny is not only on Snapchat, but actively posting information to his public "story" that we can use to pivot to other social media platforms. 

{:refdef: style="text-align: center;"}
![Snapchat suggesting my new contact](https://i.imgur.com/IFTMyug.jpg){: .imgCenter}
{: refdef}

{:refdef: style="text-align: center;"}
![Target's Snapchat story](https://i.imgur.com/Rc4UZPd.jpg){: .imgCenter}
{: refdef}

On Instagram we can quickly scan his public highlights to see that he's mentioned being in Bristol, Digbeth, "Brum" (slang for Birmingham), and "Selly" (short for Selly Oak, an area in Birmingham.) Since he mentions being in Selly with housemates, we can safely assume that he lives there.

{:refdef: style="text-align: center;"}
![Instagram story highlight](https://i.imgur.com/8pKcqkv.jpg){: .imgCenter}
{: refdef}

In his highlights, he also posts a partially redacted screenshot of a flight itinerary leaving from [UNKNOWN] and heading to Amsterdam. Since we have determined that the target lives in Birmingham, we can look up flights on that day from Birmingham to Amsterdam to find the flight number which is `KL 1426`. All that's left is to do a quick Google search for the ISO 3166-1 Alpha 2 code (which for England is surprisingly 'GB' and not 'UK.')

We put all this together to get our flag : `rgbCTF{donovanlockheart:birmingham:gb:kl1426}`

{:refdef: style="text-align: center;"}
![Instagram story highlight](https://i.imgur.com/b8PzcpG.jpg){: .imgCenter}
{: refdef}

{:refdef: style="text-align: center;"}
![Target's flight information](https://i.imgur.com/cfnG4Rh.png){: .imgCenter}
{: refdef}

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------

## Robin's Reddit Password {#robins-reddit-password}
{: style="text-align: center"}
####  Category: Forensics/OSINT | Solves: 30 | Points: 490
{: style="text-align: center"}

----------------

{:refdef: style="text-align: center;"}
![Challenge text](https://i.imgur.com/TGz3N9X.png){: .imgCenter}
{: refdef}

I have to admit that at first I couldn't quite wrap my head around this one, but the bit about breaking into Reddit's server reminded me of a funny easter egg someone had sent me before.

{:refdef: style="text-align: center;"}
![reddit easter egg](https://i.imgur.com/n7Fe1SN.png){: .imgCenter}
{: refdef}

Sure enough, one of the users here is Robin.  We google the hash and find out that someone has already cracked the hashes for us (thank you, stranger) so we can just wrap it in the flag format and submit it!

{:refdef: style="text-align: center;"}
![reddit hashes crakced](https://i.imgur.com/TZF5DWe.png){: .imgCenter}
{: refdef}

It's also worth noting that this challenge could have just as easily been solved with one Google search as well.

{:refdef: style="text-align: center;"}
![](https://i.imgur.com/KxCvY13.png){: .imgCenter}
{: refdef}

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------


## Alien Transmission 1 {#alien-transmission-1}
{: style="text-align: center"}
#### Category: Forensics/OSINT | Solves: 158 | Points: 219
{: style="text-align: center"}

----------------

{:refdef: style="text-align: center;"}
![Alien Transmission Challenge Text](https://i.imgur.com/EIH4MFK.png){: .imgCenter}
{: refdef}

We're given a .WAV file, and the clue tells us that it came over the radio, so right away we can go looking for ways to decode this 36 second audio clip into something meaningful.  When you open it with Audacity and look at the spectrogram,  we are presented with the image on the left.  After a little bit of digging, the image matches pretty closely with what a [Slow Scan Television (SSTV)](https://en.wikipedia.org/wiki/Slow-scan_television) signal looks like (image on the right) 

{:refdef: style="text-align: center;"}
![Audacity ](https://i.imgur.com/0dOxkYh.png){: .imgCenter}
{: refdef}

Often times programs that pull data from audio involve complex setups with virtual audio devices and whatnot. Luckily there's a tool on GitHub ([xdsopl/robot36](https://github.com/xdsopl/robot36)) to encode/decode SSTV in a mode called Robot36. After compiling the tool, we run it against the file and grab our flag!

{:refdef: style="text-align: center;"}
![robot36 output](https://i.imgur.com/HqFVEe8.png){: .imgCenter}
{: refdef}

{:refdef: style="text-align: center;"}
![The Flag](https://i.imgur.com/fDnVkar.png){: .imgCenter}
{: refdef}

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------

## Typeracer {#typeracer}
{: style="text-align: center"}
####  Category: Web | Solves: 184 | Points: 119
{: style="text-align: center"}

----------------

{:refdef: style="text-align: center;"}
![Typeracer challenge description](https://i.imgur.com/nPMIccf.png){: .imgCenter}
{: refdef}

Straight out of the gate, this looks like a pretty neat little game.  You can't just copy and paste the sample text because in the page source, each word is generated out of order, in a different span, and on a different layer. You could try to de-obfuscate and try to figure out the javascript that renders that whole mess, but what a pain in the ass that would be.

{:refdef: style="text-align: center;"}
![Typeracer web app](https://i.imgur.com/odLfFQE.png){: .imgCenter}
{: refdef}

{:refdef: style="text-align: center;"}
![obfuscated javascript](https://i.imgur.com/ug05QDS.png){: .imgCenter}
{: refdef}

The way Firefox Developer Tools were designed actually makes this challenge an absolute breeze. When you start typing a variable name in the javascript console, it not only brings up a list of variables, but also shows the value of whichever one you highlight. Since the obfuscation renames variables in the format of `_0xFFFFFF`, we can easily find the variables corresponding with our input textbox and the target string. 

To solve the challenge from here, all we have to do is type `_0x318f00 = _0x35179f` into the console, start the game, and hit enter.

{:refdef: style="text-align: center;"}
![Firefox Developer Tools Javascript console](https://i.imgur.com/QeVTADL.png){: .imgCenter}
{: refdef}

{:refdef: style="text-align: center;"}
![the base64 encoded flag](https://i.imgur.com/LUI3KWa.png){: .imgCenter}
{: refdef}

----------------

{:refdef: style="text-align: center;"}
[--- Back to Top ---](#intro)
{: refdef}

----------------