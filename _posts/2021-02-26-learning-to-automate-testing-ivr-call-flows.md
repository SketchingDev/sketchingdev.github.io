---
layout: post
title:  "Learning to automate testing IVR call flows"
date:   2021-02-26 00:00:00
categories: ivr call flow automation
image-base: /assets/images/posts/2021-02-26-learning-to-automate-testing-ivr-call-flows
---

We all know Interactive Voice Response (IVR) call flows as the automated systems that answer our calls when we phone a
company with an issue. Through a flurry of automated questions they seek to answer the simple queries, so agents can 
spend more time on the difficult ones.

Having recently worked on such flows I've been surprised by the lack of freely available tooling to automate interacting 
with IVR flows. I've instead been subjected to manually calling the flow with each change - a slow and erroneous
process. If there was a tool to automate this then there would be many benefits:

- Customers could get value quicker. Those developing flows wouldn't have to waste so much time manually testing them 
  (reducing [cycle time](https://www.davefarley.net/?p=218))
- [Synthetic testing](https://en.wikipedia.org/wiki/Synthetic_monitoring) could be setup to routinely test that 
  production call flows are working as expected from the customer's perspective
- End-to-end tests could test flows and their integrations as part of continuous delivery pipeline


This is why I've been creating [IVR Tester](https://github.com/SketchingDev/ivr-tester), a tool that can simulate a
human calling an IVR call flow and assert it behaves as expected. It has been an interesting experience piecing
together all the relevant technologies, so I thought it might be fun to give a high-level (hopefully non-technical)
overview of how it simulates calling a flow and traversing its menus.

To begin with let's obverse a human interacting with a call flow...

## How a human interacts with a IVR call flow

From the the customer's perspective they:
1. Call the number from their mobile or landline phone
2. Hear a (often a computer generated) voice asking a question e.g. "press 1 to make an appointment or 2 to ..."
3. Respond by either speaking a word, or more likely by pressing a key on the keypad
4. Repeat from step 2 until their query is resolved, they're forwarded to a human or they hang up in frustration

<Fix this image. Not sure what I dislike about it. It just doesn't look right>

![Human calling IVR call flow and pressing keypad in response to questions]({{ page.image-base }}/human-interaction.png)

## Simulating a human interacting with a IVR call flow

Let's tackle each step and build out our application to see how we can simulate a human calling a real IVR flow.

<p align="center">
  <img alt="Blank application ready for functionality to be added" src="{{ page.image-base }}/application-1.png" />
</p>

## Calling the IVR Flow

Calling a phone number is a complicated business, but thankfully companies like [Twilio](http://twilio.com/) abstract 
away this complexity. In Twilio's case they offer their
[MediaStream API](https://www.twilio.com/blog/media-streams-public-beta), which can call a phone number and then
establish an bi-directional audio stream with our application. Our application is then capable of listening to the call,
and responding with its own audio.

Plumbing in this functionality results in a 3 step process to establish a call:
1. Our application requests that Twilio makes the phone call for us
2. Twilio does the complicated business of making call
3. Twilio then establishes the bi-directional audio stream with our software

![Application establishing bi-directional audio stream with IVR flow]({{ page.image-base }}/application-2.png)

If you're interested you can see what the code looks like for this here:
[Source code](https://github.com/SketchingDev/ivr-tester/blob/4d85b12d4d1187072145690e70f4a6a456401119/packages/ivr-tester/src/call/TwilioCaller.ts#L40-L61)

We now have our call. The next question is how we understand what the IVR call flow is asking us...

## Hearing the IVR flow

Millions of years of evolution and a basic education means that us humans can understand the audio prompts spoken by
the IVR flow, however computers haven't been so fortunate. To provide a basic recognition capability we're going to
transcribe the audio, as text is much easier to perform matches on (which we'll do later on).

The two main providers for transcribing audio streams are 
[Google's Speech-to-Text](https://cloud.google.com/speech-to-text) or 
[Amazon's Transcribe](https://aws.amazon.com/transcribe/). Although both APIs are very similar Google's seems more 
geared towards telephony since it allows you to provide the telephony standard audio format of 8-bit PCM 
mono uLaw (8Khz sampling rate) and has a model pre-trained specifically for phone calls.

![Application transcribing audio from Twilio]({{ page.image-base }}/application-3.png)

[Source code](https://github.com/SketchingDev/ivr-tester/blob/4d85b12d4d1187072145690e70f4a6a456401119/packages/transcriber-google-speech-to-text/src/GoogleSpeechToText.ts)

Now when we establish a call we can pipe the audio stream to Google's Text-to-Speech and we eventually end up with the
transcription below. Unfortunately it is not perfect and can make mistakes...

*"Please enter the numbers from your post code. For example, If your post code is w 12 8 Q c. He would enter 12 indeed.
Please. Breath has when you are finished"*

When matching on the text to determine what to respond with we need to be lenient to these mistakes e.g. by matching on
the section of text 'Please enter the numbers from your post code'.

## Responding to a question

Navigating a call flow is more reliably performed using your phone's keypad - we've all had an unfortunate experience
with voice-recognition. Pressing your keypad works by producing a tone within the voice-frequency band that is unique
to each key. These tones are more formally known as DTMF tones (or touch-tones).

Producing them programmatically is beyond by expertise, however 
[creating audio files that can then be played back isn't](https://github.com/SketchingDev/ivr-tester/tree/4d85b12d4d1187072145690e70f4a6a456401119/packages/ivr-tester/src/call/dtmf/raw).
We can create the sound files using [Sound eXchange (SoX)](http://sox.sourceforge.net/) like so:

```shell
# Generates DTMF tone '1' by combining 1209 Hz and 697 Hz sine waves
sox -n -r 8000 -t raw -e u-law -c 1 -b 8 1.raw synth 0.5 sin 1209 sin 697 pad 0 0.5
```

![Application responding to a call with DTMF tones]({{ page.image-base }}/application-4.png)

We can now push the contents of these files onto the call's audio stream to simulate a key being pressed on a phone's
keypad.

## Determining when to play a particular response
Although we can simulate a keypad being pressed, we need some way of defining when to play them. Since these will
differ based on the IVR flow being called we don't want to them entangled into our software, so instead we'll create
an external file that we can just load in depending on what IVR we're calling.

This is an example of such an external file. It will simulate pressing '1234567' when the prompt it hears contains
'please enter your account number'. Hopefully it is fluent enough to help document the flow too.

```typescript
const test = {
  name: "IVR asks for an account number",
  test: inOrder([
    {
      whenPrompt: contains("please enter your account number"),
      then: press("1234567"),
    },
    // ...
  ]),
}
```

## Putting it all together

![Application that can establish, translate and respond to calls]({{ page.image-base }}/application-5.png)

After glueing this all together and adding some bells and whistles you end up with something like
[IVR Tester](https://github.com/SketchingDev/ivr-tester).

![Terminal running a test with IVR Tester]({{ page.image-base }}/terminal-recording.gif)
