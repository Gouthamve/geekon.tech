+++
date = "2016-02-11T23:13:17+05:30"
draft = false
title = "Controlling the Parrot AR using Gestures"
slug = "controlling-parrot-drone-using-gestures"

socialsharing = true
totop = true

banner = "/img/posts/two/cover.jpg"

author = "Goutham"
authortwitter = "https://twitter.com/putadent"

categories = ["hardware", "node", "nvision"]
+++

The winning idea of the IBM drone hackathon in nvision, 2k16

# Prologue

First of all a huge thanks to IBM for conducting a drone hackathon by giving us a Parrot AR drone and DJI Phantom to build stuff on.
As a result we developed some cool projects that will be showcased in a series of blog posts.

The first prize went to the project by Yash (https://www.facebook.com/yash.pitroda1) and Niranjan (https://www.facebook.com/sai.n.reddy1), where they made a system that controlls the Parrot AR drone via gestures.

# The Project

The project itself consists of two parts: The Hand and the nodejs program to control the drone.

## The Hand
<img src="/img/posts/two/hand.jpg" width="50%" style="float: right"></img>
This awesome hand was made by a team of 1st years as part of their course Independent Project. They originally made it to interpret hand symbols, so as to help mute people.

But it was generic enough that it could be used to detect any sign made. It was made using very economic and easily available hardware:

  1. 5x Flex Sensors
  2. 1x Accelerometer+Gyro Sensor
  3. 1x Arduino Mega
  4. 1x LCD Display

A technical description of how it works is given below:

`The device makes use of flex (resistance) sensors along with accelerometer and gyro sensor to convert hand symbols into text. The flex sensor gives a resistance value according to the folding of the finger. It gives a high value for a curled finger and low value for a straight finger. The flex sensors are connected to the arduino which in turn processes the signals along with signals from accelerometer and gyro sensor to produce a output. Different inputs are mapped to various output, which can be text, speech or anything you want.`

Now we used this hand to map several gestures to the different controls of the drone. We hooked this hand to the computer and printed the detected command into Serial, where it was picked up by nodejs program.

## node.js

We then used the excellent [node-ar](https://github.com/felixge/node-ar-drone) library by [@felixge](https://github.com/felixge) to control the drone.

The library made it so simple, that this was all the code:
```js
var SerialPort = require("serialport").SerialPort;
var serialport = new SerialPort("/dev/ttyACM0", {
    dataBits: 8,
    baudrate: 9600,
    parity: 'none',
});

var k;
var ardrone = require('ar-drone');
var client = ardrone.createClient();

serialport.on('data', function(data, callback) {
	k = String.fromCharCode(data[0])
	console.log(String.fromCharCode(data[0]));
	if (k == 't') { // takeoff
	    client.takeoff();
	    console.log("taking off");
	} else if (k == 's') { // stop or hover mode
	    console.log("hovering");
	    client.stop();
	} else if (k == 'u') { // up
	    client.up(0.3);
	    console.log("going up");
	    client.after(10, function() {
	        this.stop();
	        console.log("stopped");
	    });
	}
	// More conditions for other controls
});
```

Now these two combined to give this super-awesome demo:

<iframe width="560" height="315" src="https://www.youtube.com/embed/JmeAF-jghbY" frameborder="0" allowfullscreen></iframe>

# Epilogue

**Whats next?** 
This is the question we have been asking ourselves all the time. We first plan to make the hand wireless and then we want to use [Bluemix](https://bluemix.net) to send the drone remote commands. With that, I think we can have long range drone control using Gestures.