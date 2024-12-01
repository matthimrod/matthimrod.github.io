---
layout: post
title: Raspberry Pi Zero W "Busy Light"
subtitle: A Practical Raspberry Pi Project
gh-repo: matthimrod/busylight
gh-badge: [star, fork, follow]
cover-img: /assets/img/2021-01-25-raspberry-pi-busy-light.jpg
thumbnail-img: /assets/img/2021-01-25-raspberry-pi-busy-light-thumb.jpg
share-img: /assets/img/2021-01-25-raspberry-pi-busy-light.jpg
tags: [vpn, howto]
author: Matt Himrod
---

#### A Practical Raspberry Pi Project
![](/assets/img/2021-01-25-raspberry-pi-busy-light.jpg)

COVID-19 has us many of us working from home. My husband and I are both fortunate enough to be in this group. However, as a teacher, his schedule is very predictable. Mine, however, is not. I don't like leaving the door to my office closed all the time, and it's not always easy to tell when I'm on a call because I'm not always the one talking.

With this in mind, I had a problem to solve! I'd found some USB-based "busy light" devices, but they all had short cords. I wanted something wireless that I could mount on the wall outside my home office. I was hoping that I could make something that would automatically change color when communication activity was detected or when my Microsoft Teams status was "on a call" or "do not disturb". 

I found a few blog posts from people in similar situations, but nothing that exactly met my needs. However, I did find things that I could merge together.

First, I found this: 

[DIY: Building a busy light to show your Microsoft Teams presence](https://www.eliostruyf.com/diy-building-busy-light-show-microsoft-teams-presence/)

It was a great start. It gave me the idea to use a Raspberry Pi Zero W, but I quickly discovered that my employer has restrictions on the Microsoft Graph API that the author uses to query his Teams Presence. I also found some of the parts are discontinued, but more on that in a minute.

Second, I found this:

[DIY Microsoft Teams On-Air Light!](https://foxdeploy.com/2020/07/28/diy-microsoft-teams-on-air-light/)

This one uses a Wemo Smart Switch to turn on and off a standard light bulb. It wasn't what I wanted exactly, but it gave me an idea to solve the critical issue of querying Teams for my Presence. Instead of querying Teams, I'd query Skype for Business. My organization has SfB disabled, but the SfB app still mirrors my Status/Presence from Teams. This was key to solving my problem.

## The Hardware

First up, the actual hardware:

* Raspberry Pi Zero W (with headers)
* Pimoroni Pibow Zero W Case (PIM258)
* Pimoroni Unicorn HAT Mini (PIM498)
* Pimoroni pHAT Diffuser (PIM309)
* MicroSD Card
* Raspberry Pi Power Supply

These were all very easy to find by doing a quick search with your favorite search engine. Pimoroni has a wide array of gadgets and LED boards, so if you can't find the parts I used, I'm sure you can certainly find an alternative product and make your own variation. The pHAT Diffuser isn't a requirement, but it gives the LEDs a nice soft glow and makes the final product look a lot more polished.

The hardware was simple to assemble, so I won't go into detail on that. Mounting was also easy -- I used a couple removable stick-on hooks, but hook-and-loop tape would also work well.

##User Presence

In Microsoft Teams speak, presence is the user status. The "proper" and up-to-date way to get this is through the relatively new Microsoft Graph API, specifically the Get Presence API, but this requires the Presence.Read permission. As I mentioned before, my organization requires administrative approval to use this permission, so it's not enabled on my own user account. That would be too simple, and conveniently, it would allow me to implement this entire project without any software on my work computer. I digress...

The next best thing is to use Skype for Business with the Microsoft Lync SDK. My implementation deviated from the example I found above because I did some digging into the objects that I was querying. Instead of using the ContactAvailability Enum, I discovered that one of the other attributes in the Contact object is "ActivityId", which was slightly more descriptive. For example, it differentiates between "in-presentation" (that is, sharing my screen) vs. just "on-the-phone" (that is, just on a normal call). 

This all turned out to be very easy to put into a PowerShell script:

```
$Url = "http://busylight.lan:5000/api"
$Delay = 15

Add-Type –Path "C:\Program Files (x86)\Microsoft Office 2013\LyncSDK\Assemblies\Desktop\Microsoft.Lync.Model.dll";
$Lync = [Microsoft.Lync.Model.LyncClient]::GetClient()

$LastActivity = ""

while($true){
    $activity = $Lync.Self.Contact.GetContactInformation("ActivityId")    
    if ($activity -ne $LastActivity) {
        Invoke-RestMethod -Uri $Url -Method 'Post' -Body @{ state = $activity }
        $LastActivity = $activity
    }
    start-sleep –Seconds $Delay 
}
```

The script establishes a connection to the LyncClient and then polls every 15 seconds for my ActivityId. If it's different than the last poll, the script makes an HTTP POST to the Raspberry Pi which listens on port 5000. It runs an a continuous loop.

The final version that traps exceptions and starts Skype for Business if it's not already running is in my Github. That version also detects when Skype for Business exits, sends an "off" signal to the light, and exits the script. This is handy for when I shut down my laptop on weekends so the light doesn't stay on all the time.

Note that "busylight.lan" is the local DNS record that my router assigned to my Pi Zero. I'll get to that side of things next.

## The Light

Now, the important part: the Raspberry Pi with its LED panel!

Pimoroni publishes an API on Github for the Unicorn HAT Mini. This was another piece that started simple and evolved. Here's the basic script:

```
#!/usr/bin/env python3
  
import json
from flask import Flask, request
from unicornhatmini import UnicornHATMini 

app = Flask(__name__)

unicornhatmini = UnicornHATMini()

def set_state(text):
    if text in config['statuses']:
        return set_color(config['statuses'][text])
    elif text == 'off':
        return set_color('off')
    elif text in config['colors']:
        return set_color(text)
    else: 
        return False
 
def set_color(color):
    if color == 'off':
        unicornhatmini.clear()
        unicornhatmini.show()
        return True
    elif color in config['colors']:
        unicornhatmini.set_all(config['colors'][color][0],config['colors'][color][1],config['colors'][color][2])
        unicornhatmini.show()
        return True
    else:
        return False
 
@app.route('/api', methods=['POST'])
def set_presence():
    if request.form.get('state') and set_state(request.form.get('state').lower()):
        return '200 - OK'
    else:
        return f"Unable to process request.", 400
 
if __name__ == '__main__':
    with open('config.json', 'r') as read_file:
        config = json.load(read_file)
    set_color('off')
    app.run(host='0.0.0.0')
There's also a config file that maps the states to colors and the colors to RGB values:

{
    "colors": {
        "red":    [255,   0,   0],
        "yellow": [255, 192,   0],
        "green":  [  0, 128,   0]
    },
    "statuses": {
        "away":            "off",
        "berightback":     "off",
        "busy":         "yellow",
        "donotdisturb":    "red",
        "free":          "green",
        "in-a-meeting":    "red",
        "in-presentation": "red", 
        "on-the-phone":    "red"
    }
}
```

This script reads the values from the JSON file, starts a Flask micro web framework instance, and creates a simple API endpoint. It takes parameter "state" whose value corresponds to the ActivityId from the Lync API. It can also take the name of a color instead of the status.

The final version of this is also in my Github. I've added a lot of features such as a web preview of the light's state, a web-based manual control, a brightness and default state settings, and a temporary override. I've also created a Tasker profile and task that changes the light to red when I'm using my cell phone!

This project was definitely fun, and my husband has told me that it's handy having the light so that he doesn't interrupt me. 🙂
