---
layout: post
title:  "Conquering the Monolith beast with Android MonkeyRunner"
date:   2015-08-03 00:00:00
categories: android monkeyrunner
image-base: /assets/images/posts/2015-08-03-conquering-the-monolith-beast-with-android-monkeyrunner
---

Though it may be generally considered cheating, when I was faced with having to endure repeatedly tapping the beast in the game Monolith on Android - since renamed to [Tap & Conquer](https://play.google.com/store/apps/details?id=com.cac.tap.and.conquer) - I did what every good programmer does and automated the task.

The tool I used was [Android's MonkeyRunner](http://developer.android.com/tools/help/monkeyrunner_concepts.html) as I was planning on using to test the UI of an app I was writing, so provided me with good practice.

![Autotapper script in action]({{ page.image-base }}/script-in-action.png)

## Setting up MonkeyRunner

Monkey Runner is part of Android's Software Development Kit (SDK). Follow the instructions in the links below to download the SDK and configure your device
to enable development mode.

1. Follow the official [instructions to install the standalone Android SDK tools](https://developer.android.com/sdk/installing/index.html?pkg=tools).
2. Follow the official [instructions for enabling ADB debugging on your device](http://developer.android.com/tools/help/adb.html#Enabling).

## Running the autotapper

1. Download the script for the autotapper (below) to your computer
2. Plug in your Android Device via the USB
3. Start the game, ensuring that the beast is showing
4. Run the script via the command below:

```bash
$ ~/adt-bundle/tools/monkeyrunner ~/monolith_autotap.py
```

You can customise the position of the taps by changing the co-ordinates in the line

```python
tap_positions = [[200, 400], [300, 410]]
```

### Autotapper script

```python
#! /usr/bin/env monkeyrunner

from com.android.monkeyrunner import MonkeyRunner, MonkeyDevice
import sys
import signal
import random

# Automate the tapping of the beast in [Monolith](https://play.google.com/store/apps/details?id=com.cac.monolith)
# by using MonkeyRunner.
#
# Read the instructions for setting it up here:
# https://sketchingdev.co.uk/blog/conquering-the-monolith-beast-with-android-monkeyrunner.html
#
# ##### "Tapping is so Middle Ages!"
# #####   -- Monolith
# #####
# #####

__author__ = 'Lucas'
__homepage__ = 'https://sketchingdev.com/'
__version__ = '1.0'
__date__ = '2015/08/03'

device = None

def execute():
    print "Connecting to device..."

    device = MonkeyRunner.waitForConnection()

    print "Connected"

    tap_positions = [[200, 400], [300, 410]]
    tap_sleep_time = 0.09

    old_phone_rest_time = 5
    old_phone_sleep_interval = 1000
    old_phone_sleep_counter = 0

    print "Tapping..."

    while True:
        for tap_position in tap_positions:
            device.touch(tap_position[0] + random.randint(1, 20),
                         tap_position[1] + random.randint(1, 20),
                         'UP')

        MonkeyRunner.sleep(tap_sleep_time)

        old_phone_sleep_counter += 1
        if old_phone_sleep_counter == old_phone_sleep_interval:
            print "Letting my old phone have a %s second rest" % old_phone_rest_time
            MonkeyRunner.sleep(old_phone_rest_time)
            old_phone_sleep_counter = 0


def exit_gracefully(signum, frame):
    print "Gracefully exiting"

    signal.signal(signal.SIGINT, signal.getsignal(signal.SIGINT))
    if device is not None:
        device.shell('killall com.android.commands.monkey')
    sys.exit(1)


if __name__ == '__main__':
    signal.signal(signal.SIGINT, exit_gracefully)
    execute()
```

## Stopping the autotapper

Press `Ctrl+C` to stop the autotapper script, this will allow the script to clear up gracefully.

## Possible problems

The following are a few workarounds to little problems I ran into whilst writing the script but didn't get around to finding permanent solutions.

### Autotapper fails to connect to device
If the script doesn't manage to connect then it could be that the ADB daemon needs to be started. This can be achieved by running the following.

```bash
$ ~/adt-bundle/platform-tools/adb start-server
* daemon not running. starting it now on port 5037 *
* daemon started successfully *
```

### Autotapper keeps throwing 'Error sending touch event'

I found that approximately 1 in every 2 times I ran the script it would fail with the error below. I don't know why it does this but it is easily remedied by stopping the executing of the script (via `Ctrl+C`) and then running it again.

```
150803 09:12:56.303:S [MainThread] [com.android.chimpchat.adb.AdbChimpDevice] Error sending touch event: 209 409 DOWN_AND_UP
150803 09:12:56.303:S [MainThread] [com.android.chimpchat.adb.AdbChimpDevice]java.net.SocketException: Broken pipe
```
