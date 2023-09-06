---
title: Over Complication
date: '2022-29-11'
tags:
  - automation
  - blog
---

Reading through Mastodon this morning, I came across [a fun post](https://sixcolors.com/post/2022/11/creating-a-smart-on-air-sign-with-an-e-ink-display/) by Dan Moren, which sparked some ideas for me of ways to massively over-complicate simple tasks. In this age of remote work, I spend _too much_ time on conference calls, at various hours of the day, and while my low tech approach of 'if the door is closed, don't come in' is moderately successful, what I really like doing is spending time and money on techy stuff, so it's time to play!

My first port of call is [Home Assistant](https://www.home-assistant.io). I've been slowly building up the smarts in the house, and it's a great centralised point to gather information, display it, or act on it in a whole range of ways. If I can get my 'on air' status into Home Assistant, the sky is really the limit on what I can do - could turn lights on (or change their colour), announce on my Echos that "Dad's on call", send notifications to people - any number of wacky and wild things that are totally over the top.

The easiest approach to this is to set up a `binary sensor` - this is a simple entity that has an on/off state, and is perfect for my needs - I'm either on air, or not, there's no need for extra state (or is there...?). These are pretty easy to set up in Home Assistant, but then the over complications started to swirl. "What if I want to change the state remotely?" "Should I be tied into Home Assistant for everything?" "Is there a more complicated way to do this"?.

I have an MQTT Broker set up that is running a number of things in the house. Not only do an ever growing cache of Zigbee devices use it (via the _excellent_ [zigbee2mqtt](https://www.zigbee2mqtt.io)), but I also have my Flight Tracking Raspberry Pi send its data to the Message Bus too. The other beauty is that I could decouple things from Home Assistant if I wanted, and have physical devices both respond to, and cause the status change, so it rapidly became a no brainer.

Diving into the docs, it turned out that setting up a sensor that runs of MQTT is quite straightforward. Simply adding the following in my `configuration.yaml` - the default home for lots of config did the job:

```yaml
mqtt:
# MQTT-Based Sensor to say if I'm on a call
binary_sensor:
  - name: Office On Air
    unique_id: 3de52c93-5be8-46sb-5aab-rf2fe8306921
    state_topic: 'homeassistant/office/onair'
    payload_on: '1'
    payload_off: '0'
    qos: 0
    device_class: running
```

The above will listen for messages on the `homeassistant/office/onair` topic, with a payload of '1' or '0' and be on or off depending on which one is sent.

icon_color: '#3AA2E1'
58 162 225 100
color: rgb(58,162,225)

badge_color: '#44739E'
