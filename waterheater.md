## Hacking your water heater for fun and profit
Ok, no profit.

I love home automation and have my [airbnb fully automated](https://www.motoretreat.com/automation).   In mid November 2022, my water leak sensor detected a leak near my hot water heater and automatically shut off the water to the house.   A 3 hour drive later, I discovered my old water heater had failed pretty spectacularly.

After some research, I decided to buy a Rheem 80 gallon hybrid water heater (traditional heating elements + heat pump) to replace it.  I had already used a GE 220v/40A ZWave smart switch to turn my old water heater on/off based on the Airbnb calendar, but this Rheem unit has wifi and can be put into 'vacation mode', which is better because it keeps the water at 65ºF/18ºC, out of freezing danger in the winter months.

I [took over a non-maintained Hubitat Rheem integration](https://github.com/klinquist/hubitat-rheem) and began exploring their API.   Most of the hard work was done by others prior, I just wanted to fully understand how it worked.  

Rheem uses the [ClearBlade](https://www.clearblande.com) platform and have both an HTTP and an MQTT API.   I was able to use CharlesProxy on my iPhone to man-in-the-middle the HTTP API to understand how to get a bearer token (a JWT) as well as capture the webhooks the phone sends when you enter/leave a geofence to put your devices in away mode.

ClearBlade documentation told me about user and device topics.   With my bearer token, I was able to publish to the topic `user/{userId}/desired` (desired state - where requests are made) and subscribe to the topic `user/{userId}/reported` (reported state - where acknowledgements happen).  I was already familiar with this concept as I have extensive experience with AWS IoT Core, which uses similar nomenclature. 

I tried subscribing to `#` (MQTT wildcard)...and immediately got a flood of data that included:

* Emails any time a user logged in or out of the app
* All device information including temperature sensor statuses, owner zip code, and mode (vacation mode!)

**I immediately contacted anyone at Rheem that sounded IT/Platform/Security related on LinkedIn  - as well as their security compliance email -  and told them I need to discuss a serious vulnerability in their API - and that I had access to a list of customers that logged into their app.**  

I received an email from their compliance department that they would look into it and get back to me.

In the mean time, I wrote a little service that opened up a subscription to `#` and filtered on my device information, which is published to the topic `device/{mac_address}/{serial_number}/{model_id}/reported`  (I did not have permissions to subscribe directly to my device topic).

### I actually hit my Comcast data cap because the MQTT stream was ~130GB/day worth of data.

I was able to get lots of neat information: Compressor run time, kWh consumed, tank temperature..  There were a dozen or more parameters:

```
      "FAN_CTRL": 0,
      "COMP_RLY": 0,
      "EXACTUAL": 100,
      "HRSLOFAN": 35.308345,
      "HRS_COMP": 97.127822,
      "WHTRENAB": 1,
      "WHTRSETP": 120,
      "HOTWATER": 60,
      "HRSHIFAN": 61.998172,
      "PRODDESC": "Heat Pump Water Heater          ",
      "PRODSERN": "XXXXXXXXXX                      ",
      "UNITTYPE": 4,
      "PRODMODN": "XE80T10H+45U0                   ",
      "SW_VERSN": "WH-HPW5-H6-01-04",
      "SERIAL_N": [
        "XXXXX"
      ],
      "UPELSIZE": 4.5,
      "LOELSIZE": 4.5,
      "LSDETECT": 0,
      "HEATCTRL": 0,
      "SHUTOFFV": 0,
      "SHUTOVER": 0,
      "ANODEA2D": 42405,
      "ANODESTS": 2,
      "SHUTOPEN": 0,
      "SHUTCLOS": 0,
      "HRSUPHTR": 2.938611,
      "HRSLOHTR": 0.002222,
````

Many of the key/values I didn't understand, so I did what I usually do: Search github!


Doing that, I found something even more alarming:  A public repository that appeared to be all of their Python QA test automation scripts.   This included READMEs, test accounts, and *the password for admin@rheem.com*.   

This can't be a user in their production environment, can it? 

Yes, it can. I successfully logged into their app.


A week goes by, and I notice that I can no longer subscribe to `#`.  Ok, someone investigated after I sent the LinkedIn messages and closed that hole.

So I went back to ClearBlade documentation, and found they have another "trigger" topic that gets published to for "user:login", "device:create", etc.

What do you know, any user still has access to subscribe to `$trigger/#` and receive all of that information.


Almost 2 weeks had passed since I received the "we'll look into it" reply,so I replied again saying that I'd really like to talk, and this time I attached a list of emails who had launched the app in the past 10 minutes.

That got their attention.. within hours, I lost access to `$trigger/#`.  We scheduled a call.

In the meantime, I grabbed a bearer token for the admin@rheem account to see if it had any additional MQTT permissions.

Oh yeah.  It did.

I could publish or subscribe to any topic.  That means I could publish payloads to `/desired`, changing the settings on anyone's water heater or thermostat.  I really should have just renamed everyone's devices to **pwned.**


Today, I had a call with Rheem, and talked about how I discovered what I did and gave them several recommendations. I told them I didn't want anything in return.  I did request that they grant a user access to their own device's `/reported` topic, for nerdy data-gatherers like myself.



Obviously, that admin account password was changed by Rheem shortly after I got off the call with them. JWTs have been revoked, but for now, my MQTT connection capturing my water heater stream is still open.  



I wonder how much of the Rheem API github copilot will autocomplete :).