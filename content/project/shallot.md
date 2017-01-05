+++
title = "Shallot"
description = "WebRTC-powered peer-to-peer onion routing."
image = "projects/shallot.png"
report = "/docs/shallot.pdf"
github = "https://github.com/FelixMcFelix/Shallot"
categories = ["Networks", "University"]
date = "2016-03-28"
tags = ["JavaScript", "NodeJS"]

+++

Shallot implements a basic decentralised onion routing scheme designed for use within browser applications, by using a modification of the *Chord* protocol as a backbone network and file-storage system.
This is built upon the experimental *WebRTC Data Channels* recently added to browsers to offer true peer-to-peer connectivity between users.
The system is compatible with both NodeJS and browser-based environments, relying upon one server node as an entry-point.
Shallot was my Honours project---the above link directs to the [project dissertation](/docs/shallot.pdf).

<!--more-->

At the lowest level, the *Conductor* library allows for simple connection management, designed for use in peer-to-peer systems which allow nodes different ways to communicate with one another.

Above this, a modification of the Chord peer-to-peer protocol provides best-effort message delivery between clients and uses a secure scheme for client authentication.
The traditional features of Chord are then implemented over this messaging protocol, with some modifications appropriate for an adversarial network.

The onion routing layer is simply another module within the Chord system, making use of the file system and certification schemes it provides.