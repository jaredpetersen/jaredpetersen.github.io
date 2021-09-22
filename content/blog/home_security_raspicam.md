---
title: How to Set up a Home Security Live Streaming Camera with Raspberry Pi
slug: home-security-raspicam
description: Tutorial for how to set up a home security system using the Raspberry Pi, Raspberry Pi camera module, and raspilive
date: 2018-08-16T00:00:00-07:00
---

![Raspberry Pi Camera connected to the Raspberry Pi Camera Module V2](/img/home-security-raspicam/raspberry-pi-camera.jpeg 'Image courtesy of the Raspberry Pi Foundation, CC BY-SA 4.0, via Wikimedia Commons')

**UPDATE: This project has been rebranded to "raspilive" and has been completely rewritten in GO. The functionality discussed here still exists but the commands have been adjusted. For updated information, please see the [project page](https://github.com/jaredpetersen/raspilive).**

One of the first things my wife and I wanted to do when we bought our first house was set up a home security system. I didn’t want to pay a monthly fee for a security service and thought it would be fun to throw Raspberry Pis with cameras all over the house.

All they needed to do was sit around and serve live video footage with a 24 hour buffer so that it could be reviewed in the case of a burglary. Motion detection would be rendered useless by my German Shepherd “patrolling” the house and since I wanted to build my own smart home control panel, any software that bundled in a user interface wouldn’t be any good to me either.

I was already going the DIY route so I thought it would be fun to write my own.

Two years later and I’ve finally gotten around to it...

## raspi-live
raspi-live is a Node.js command-line interface that serves live video from the Raspberry Pi Camera Module over the web via HLS or DASH — you choose.

![Terminal demonstration of running the help command](/img/home-security-raspicam/demo.gif)

There’s a small bit of setup involved, especially if you’re starting from a fresh operating system installation. Let’s dig in.

### Install
You’ll want to start by installing FFmpeg, a popular video processing tool used to convert the video stream coming out of the Raspberry Pi Camera Module into something that can be streamed over the internet.

Run the following commands to download and configure FFmpeg:

```sh
sudo apt-get install libomxil-bellagio-dev
wget -O ffmpeg.tar.bz2 https://ffmpeg.org/releases/ffmpeg-snapshot-git.tar.bz2
tar xvjf ffmpeg.tar.bz2
cd ffmpeg
sudo ./configure --arch=arm --target-os=linux --enable-gpl --enable-omx --enable-omx-rpi --enable-nonfree --extra-ldflags="-latomic"
```

If you’re working with a Raspbery Pi 2 or 3, run `sudo make -j4` to start the FFmpeg build process. If you're working with a Raspberry Pi Zero, run `sudo make` instead. The extra option is just to take advantage of the CPU cores we have available. After that finishes, run `sudo make install` regardless of the model of your Raspberry Pi.

FFmpeg has now been installed. Let’s clean up by deleting the `ffmpeg` directory and tar file that were created during the installation process.

All that’s left for the installation process is to install Node.js and finally install raspi-live via `npm install -g raspi-live`.

### Streaming
Now that the boring stuff is out of the way, we can start streaming.

![Terminal demonstration of running the start help command](/img/home-security-raspicam/help-command.gif)

If you’re okay with the default configuration, running `raspi-live start` is sufficient. However, there are options as well that help spice things up. You can change the output directory (maybe to a RAMDisk to help prolong your Pi’s lifespan), streaming format, number of archived streaming files kept on disk, etc. to enable you to set up your Raspberry Pi live video feed the way you like it.

Let’s run `raspi-live start` to try it out:

![Terminal demonstration of streaming video using the start command](/img/home-security-raspicam/start-command.gif)

Great! We’re live! Let’s check out the feed.

There’s a variety of ways to play the stream, but we’ll test it out using VLC since it’s a quick way to get up and running. Launch VLC and open up a network source. Put in the IP address of your Raspberry Pi along with the port number `8080` and the endpoint `/camera/livestream.m3u8`. If you decided to specify DASH using the options instead of the default HLS streaming format, you’ll need to put in `/camera/livestream.mpd` as your endpoint.

![Screenshot of VLC Source network configuration with the URL 'http://someipaddress:8080/camera/livestream.m3u8'](/img/home-security-raspicam/vlc-source.png)

Click the Open button and you’ll start seeing what your Pi sees.

![Footage of a ceiling fan that was captured by the camera](/img/home-security-raspicam/captured-footage.gif 'A little bit of artifacting is present here due to the "gif-ification" of the live stream')

By default, raspi-live outputs 720p 25fps video for streaming. While the Camera Module can output up to 1080p 30fps and 720p 60fps, you have to be careful about how much data you try to shove out of your Raspberry Pi. The video is compressed by the server via gzip/deflate to help out with that problem, but you’re still limited by the upload speed provided by your home internet provider.

Your mileage may vary, so I made it easy to configure factors like the video resolution, frame rate, compression level, etc. so that you can tune your Pi to the way you like it.

If you have any feature requests or any problems using raspi-live, open an issue on [GitHub](https://github.com/jaredpetersen/raspilive) and/or submit a pull request. I’d love to hear about others’ experiences using the software.
