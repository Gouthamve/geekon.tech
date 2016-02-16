+++
date = "2016-02-13T01:23:52+05:30"
draft = false
title = "Realtime Atmospheric Visualization on the DJI Phantom"

banner="/img/posts/three/lpg.jpg"

socialsharing = true
totop = true

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["hardware", "node", "nvision"]
+++

My Submission to the IBM drone hackathon in nvision, 2k16
We won second yo!

# Prologue
This is a blog series on the IBM drone hackathon conducted at nvision, 2k16! A huge thanks to IBM for giving a DJI Phantom 3 and Parrot AR 2.0 to hack on. I previously wrote about the winning idea [here](https://geekon.tech/post/controlling-parrot-drone-using-gestures/).

Coming to the idea implemented by me and [Lalith](https://www.facebook.com/profile.php?id=100006010906759), we built a realtime atmospheric visualization solution using the DJI Phantom. We basically put a bunch sensors and flew the drone around and mapped several factors with the altitude.

# The Project

## Hardware

We needed a stack where we could could connect a bunch of sensors to WiFi. Now while the obvious answer seems to be the Raspberry Pi, we couldn't use it because the sensors available to us were analog and the support for the common sensors sucks big time in the Raspberry Pi. And the WiFi support is extremely bad on the Arduino. **ESP8266** exists but its bad if you are trying to get a project up in 10 days.

The [Particle Photon](https://store.particle.io), is the perfect solution in cases like this. Its an awesome WiFi dev platform that is Arduino compatible. We had three photons around and we could quickly get a PoC up and running in less than a day! But as we started to add more sensors to it, we started seeing issues.

So we hooked up an Arduino Nano to the sensors and then the nano printed into the serial of the Photon which then started to beam the data up. We could send all the data to requestb.in and then we started on the software stack while Lalith was improvising the hardware.

## The WebPortal

Now this is where most of my effort into. I wrote an express.js and socket.io server which would capture the data and display the data in realtime.

I used Bluemix to deploy some of my older projects (They give a ton for free resources) and its almost like heroku! You just need to setup your code and do
`cf push <appname>`, boom your app is live at appname.mybluemix.net You can also add lots of free services from the Bluemix Marketplace.

The only issue I had with Bluemix is that you need to push your entire app every single time. Not just the git diff like in heroku. @CloudFoundry, fix this please? Also, the default node.js buildpack comes with v0.12, but it was easy to get the beta buildpack with node v4 (<3 => functions) installed instead.

**Update**: The default buildpack is now v4. Kudos!

### Cloudant

I wanted to have persistence and needed a database. I almost always use Mongo for my projects, but I had initially started out with couchdb and I think its MapReduce views are perfect for IoT, so I decided to go with cloudant as the DBaaS provider this time. And trust me, my decision has been perfect. Using cloudant is a breeze, inserting new docs is as easy as `db.insert(doc, id, callback)` and the best part is you query on MapReduce views.

This was particularly useful for this context as we were sending a doc that looks like this:
```js
{
  altitude: 19, //relative altitude
  smoke: 2200,
  lpg: 1010,
  timestamp: new Date()
  //more fields
}
```

now I needed to sort by altitude and also needed data like this:
```js
[{
  19: {
    smoke: 2200,
    lpg: 1010,
    //rest
  }
}]
```

Now in Mongo, you can still achieve it but nothing feels as easy and fast as in couch. You define a simple view
```js
if (doc.altitude) {
  emit(doc.altitude, doc) // Entire doc need not be emitted.
}
```
And boom! sorting, limiting and everything else you need is setup!

### The Server

I wanted to load initial data from couch and then use sockets to update the charts (A huge shoutout to [**plot.ly**](https://plot.ly/javascript/) for their amazing OSS charting library) in realtime.

I decided to use socket.io for the realtime updates and express.js for the server. It was a simple setup, made possible by the beautiful [plotly.js](https://plot.ly/javascript/). And with that, we were set. Deployment was again a simple `cf push dronetime`.

You check it out right now at https://dronetime.mybluemix.net

# Epilogue

This is not the end. And this is certainly not the perfect state of the project. The sensors we used were commonly available Chinese ones and they are not really accurate. We want to use better sensors and get more accurate results. Also because of all the dust in Kandi and the drones rotors, the readings are fluctuating a lot as you can see.

Then we build a good dashboard and go out checking all the polluted areas near Kandi :) . But before that, just going to fly the drone around and show off :D
