---
title: How to Set up a Home Security Live Streaming Camera with Raspberry Pi
slug: home-security-system-raspberry-pi
date: 2018-08-16T00:00:00Z
description: Tutorial for how to set up a home security system using the Raspberry Pi, Raspberry Pi camera module, and raspilive
---

# How to Set up a Home Security Live Streaming Camera with Raspberry Pi
One of the first things my wife and I wanted to do when we bought our first house was set up a home security system. I didn’t want to pay a monthly fee for a security service and thought it would be fun to throw Raspberry Pis with cameras all over the house.

All they needed to do was sit around and serve live video footage with a 24 hour buffer so that it could be reviewed in the case of a burglary. Motion detection would be rendered useless by my German Shepherd “patrolling” the house and since I wanted to build my own smart home control panel, any software that bundled in a user interface wouldn’t be any good to me either.

I was already going the DIY route so I thought it would be fun to write my own.

Two years later and I’ve finally gotten around to it...

## raspi-live
raspi-live is a Node.js command-line interface that serves live video from the Raspberry Pi Camera Module over the web via HLS or DASH — you choose.

There’s a small bit of setup involved, especially if you’re starting from a fresh operating system installation. Let’s dig in.

### Install
You’ll want to start by installing FFmpeg, a popular video processing tool used to convert the video stream coming out of the Raspberry Pi Camera Module into something that can be streamed over the internet.
