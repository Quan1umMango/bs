---
date:  2276-03-03
title: ABitTorrent Client 
description: "Recollecting some stuff about something lol idk"
layout: page 
collections: ["post", "C", "BitTorrent", "Networking"]
---

# BitTorrent
So a while back I was reading the networking book by Kurose, and I stumbled upon the BitTorrent protocol. Well I'd known about it's existence, but having never found a use for it, I never bothered to learn what it was.\
\
But while reading this book I was pretty intriguied about P2P networks, so I decided, why not make a very simple BitTorrent protocol? \
\
I wrote the program in C, for no good reason other than I find programming in it very fun (except for all the times where the program randomly crashes for some reason).\
\
This isn't a "How To Make A Client" but rather what I did to make one. Either ways, I'll be explaining the concepts in a very brief way. The project only handles v1 and single file torrents (for now).\ 
The code is freely available [here](https://codeberg.org/Quan1umMango/MinTC)

## B-encode, not Bencode
If you have used a BitTorrent client before, one of the first things you would do is to download a ``.torrent.`` file. This is the metadata for the torrent, which is encoding in Bencode (pronounced Bee encode, not Bencode which I personally think is much more fun to say).\
\
You can find the specs for this format either [here](https://www.bittorrent.org/beps/bep_0003.html) or in its [Wikipedia page](https://en.wikipedia.org/wiki/Bencode)\
\
So the first thing I had to do was write a Bencode parser. The rules are pretty simple:
- Integers are represented like this ``i<digits>e``
- String are encoded as ``<number_of_chars>:<characters>``
- Lists as ``l<elements>e``
- Dictionaries as ``d<elements>e``
## 
There are a couple of special cases, which I'll omit for the sake of brevity.\
##
Decoding the file (which is a dictionary) gives us these couple of key-value pairs:
- ``announce``: Containing the URL of the tracker
- ``info``: Contains a lot of useful such as piece lengths, total file length, SHA1 hash of each piece\
##
The next step is to query the tracker, and ask for a list of peers

## Step 2: Politely ask the tracker for peers
I've mentioned that BitTorrent is a Peer to Peer protocol. Unlike the traditional Client-Server architectures (where there is a central server which you query and get information from), in P2P apps there is no such centralized server which gives you the data. What you have are independent peers, which share the data among themselves. I'll spare you the details, but one thing the astute reader might ask is "how do i connect to these peers?". Or rather, "where can i find a list of peers?". The astute reader might once again look at the subsection title and tell you its the tracker\
\
The tracker contains a list of peers. You ask for it (passing the appropriate params)

