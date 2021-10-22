---
layout: blog-post
title:  "Hacking together a cheap realtime power usage monitor"
date:   2016-04-06 00:00:00
categories: ssocr linux fswebcam
image-base: /assets/images/posts/2016-04-06-hacking-together-a-cheap-realtime-power-usage-monitor
---

I'm currently in the throws of hacking together a bunch of old laptops into a [beowulf cluster](https://en.wikipedia.org/wiki/Beowulf_cluster), so when I wanted to monitor the cluster's power usage on my computer instead of paying out for a new-fangled gadget I did what I've been doing for the cluster and delved into my box of old electronics.

After a bit of rummaging (and online searching) I managed to bodge together the following:

* Power Meter Adaptor (LCD)
* Webcam (EyeToy for PlayStation 2)
* [fswebcam](http://www.sanslogic.co.uk/fswebcam/) - Captures webcam images
* [ssocr software](https://www.unix-ag.uni-kl.de/~auerswal/ssocr/) - Seven-segment digit recognition

![Webcam attached to power usage monitor]({{ page.image-base }}/rig.png)

The assembly of the contraption was fairly straight forward. I securely attached the webcam to the power meter via a cardboard frame, extended the webcam's LED to light the power meter's LCD and connected the whole thing to a computer with Ubuntu (which comes with the [UVC driver](https://help.ubuntu.com/community/UVC)).

## Capturing the LCD readout

Capturing an image of the display was simple enough with [fswebcam](http://www.sanslogic.co.uk/fswebcam/), which I'd configured to:

1. Skip the first 100 frames (`-S 100`) to give the webcam time to adjust its exposure levels
2. Combine 5 photos taken in quick succession (`-F 5`) to reduce the noise

```bash
$ fswebcam -r 640x480 -F 5 -S 100 --no-banner --save ~/webcam.jpg
```

## Recognising the digits

The obvious problem with using a segmented digital display is the recognition of the seven segment digits, however there is a great program called [SSOCR](https://www.unix-ag.uni-kl.de/~auerswal/ssocr/) that does this for you - and even better it also has options to help prepare the image.

```bash
$ ssocr -t 10 -d -1 rotate 3 crop 255 234 200 140 erosion ~/webcam.jpg
```
*The options used in the command are listed on [SSOCR's homepage](https://www.unix-ag.uni-kl.de/~auerswal/ssocr/).*

![SSOCR image manipulation of LCD screen]({{ page.image-base }}/ssocr-progress.png)

## Streaming the output

By automating the aforementioned tasks the following bash script stores the readings in a CSV file, which my cluster's [Dashing](http://dashing.io/) dashboard can then [tail](https://en.wikipedia.org/wiki/Tail_%28Unix%29) and display as a colourful graph.

```bash
#!/bin/bash
INTERVAL=5
WEBCAM_IMAGE="/home/lucas/webcam.jpg"
CSV_FILE="/home/lucas/data.csv"

SSOCR="/home/lucas/ssocr-2.16.3/ssocr"

while true; do
    sleep $INTERVAL

    fswebcam -r 640x480 -F 5 -S 100 --no-banner --save $WEBCAM_IMAGE > /dev/null 2>&1

    READING=$("$SSOCR" -t 10 -d -1 rotate 3 crop 255 234 200 140 erosion $WEBCAM_IMAGE)

    echo "$(date +%s), $READING" >> $CSV_FILE
    echo "$(date +%s), $READING"
done
```
