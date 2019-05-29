# Node Phea

An unoffcial [Phillips Hue Entertainment API](https://developers.meethue.com/develop/hue-entertainment/) library for Node.js. The goal of this library is to encapsulate the Hue Entertainment API while leaving a developer to use a simple color API to control color state... More documentation coming soon, but this is currently working if you already have an entertainment group, username, and psk setup. 

##### Current Version: 0.7.9

#### Features:
- DTLS Communication Setup + Messaging for Phillips Hue Bridge Entertainment API.
- Multi-light capabilities to tweek/tween lights individually, or all at once.
- Easy to use (3-command) API, while strict, allows the Hue Entertainment API to be abstracted away.
- Performance so far seems pretty good at 50Hz. There's room for improvement, but I want to hold off optimization until >1.0.0.
- Total computation time needed for 10-lights per frame is <0.1ms (~60 microseconds).

This is still a work-in-progress. That being said, the front facing API will likely not change, so it's now currently
safe to develop around Phea.

#### To-Do(s):
- Reduce reliance on async methods.
- Proper Error Handling.
- Guide or library for setting up Hue Entertainment Secrets + Groups.

## Installation:
```
npm install phea
```

## Basic Example:
```javascript
const Phea = require('phea');

let running = true;
process.on("SIGINT", function () {
    // Stop example with ctrl+c
    console.log('SIGINT Detected. Shutting down...');
    running = false;
});

async function run() {

    let config = {
        "ip": "192.168.1.10",
        "port": 2100,
        "username": "",
        "psk": "",
        "group": 1,
        "numberOfLights": 1,
        "fps": 50
    }

    let phea = new Phea(config);
    let rgb = [0,0,0];
    let tweenTime = 1000;
    
    await phea.start();

    phea.texture(lights=[], type='sine', duration=2000, depth=40);

    while(running) {
    
        rgb[0] = (rgb[0] + 85) % 256;
        rgb[1] = (rgb[1] + 170) % 256;
        rgb[2] = (rgb[2] + 255) % 256;

        await phea.transition(lights=[], rgb=rgb, tweenTime=tweenTime, block=true);
    
    }

    phea.stop();

}

run();
```

## API

#### Phea(options)
The creation of this object sets up the control ecosystem for Phea. The main parts of this ecosystem are the Hue Bridge
controller and the lights which maintain transition math. The core is called the Phea Engine where we strap all the parts 
together and put it behind an API. On top of that API, we added another API for up-front input checking for inputs that are
going to cause failures. Here, I have elected to throw errors, so bugs can be worked out sooner than later.

The following options must be provided in the instance configuration dictionary except when marked with a default value: 

##### ip (str):
The IPv4 address of the Hue Bride that will be serving the Entertainment API.

##### port (int) [default=2100]:
The Hue Bridge port listening for commands. Default is for the Hue Bridge is 2100.

##### username (str):
The 40-character string generated by the Hue Bridge server. You'll typically have to set this up manually through 
the Bridge's REST API. In the future, the goal is the automate this generation in to the app itself. 

##### psk (str):
The 32-character hex string generated by the Hue Bridge server. You'll typically have to set this up manually through 
the Bridge's REST API. In the future, the goal is the automate this generation in to the app itself. 

##### group (int):
This is the entertainment group that you will be controlling fron the Phea API. This group can be made within
the Philips Hue app. I'm considering building this into the API.

##### numberOfLights (int):
The number of lights within the entertainment group provided. The immediate goal is to automate this away. The API should
tell us how many lights are in the group.

##### fps (int) [default=50]:
This value controls the update rate of color change events to the Entertainment API. By default, this value is set to
50, which is the recomended max rate by Philips. Keep in mind, the lights themselves are limited to about 12.5fps in
real life, so that'll be your ultimate cap of light performance. The upside to this limit is that gives you somewhere in 
between 20-80ms to generate your next color. Typically, that should be pretty good times compute wise. At some point, we'll 
start looking to optimize out the library.

#### Phea.start()
This opens the DTLS socket with the Hue Bridge, then kicks the rendering loop into action. While the engine is
hidden behind the API, the engine itself is always throwing renders to the Bridge every (1000/fps) milliseconds. 
This is done to keep the light completely in-sync as well as maintain an open connection with the Bridge. This returns
a promise waiting for the establishment of the socket to the Bridge. You should wait for this to happen before any
other communication happens.

#### Phea.stop()
Shut it all down, exit the render loops, release the socket. Sometime a frame can hang and it'll take a second or two to 
fully return. I'm looking into this, but its a minor inconvienence.

#### Phea.transition(lights=[], rgb=[0,0,0], tweenTime=0, block=false)
Phea color changes are facilated through the transition method. Every color change, on every light, consist of a timeout 
loop to handle the math of updating and rendering colors to its respective light. These timeout loops can be interrupted and
replaced at any point, even immediately. This allows transitions to be set in real-time to control the lights. Keep in mind, even if tween time is set to 0, you'll still ultimately be limited by the Hue Bridge 12.5hz update rate.

##### lights=[]:
The light channels to send this color transition to. The default ([]) selects all of the lights (options.numberofLights).
One distinction with this API is that light channel input numbers, numbering starts at 0, where as, the Hue Bridge starts
numbering at 1. So when placing your lights channels, recall n-1, to place the value. 

##### rgb=[0,0,0]
This is the color the you want the transition the selected lights to. This is a 3-value array where each value is a 
uint8 representing the red, green, blue color bytes respectively. This color will be tweened to over the duration of
tweenTime.

##### tweenTime=0
This is the amount of time in milliseconds that the transition will take to tween from the existing color to
the new color. Default is 0, which means that the color change will occur on the next update to the Hue Bridge.
I don't see any performance concerns yet, but I'm trying to start finding ways to do profiling. Though, with an update
rate at 50fps, you're still looking at 15-20ms to render you're next color setting. That it typlically ample, and if not,
the framerate can be lowered to accomodate.

##### block=false
If you're calling the transition in a synchronous piece of code, set block to true, and the transition will
hold execution for the duration of the light transition. 

#### Phea.texture(lights=[], type, duration, depth)

##### lights=[]:
The light channels to send this color transition to. The default ([]) selects all of the lights (options.numberofLights).
One distinction with this API is that light channel input numbers, numbering starts at 0, where as, the Hue Bridge starts
numbering at 1. So when placing your lights channels, recall n-1, to place the value. 

##### type: 
Either 'sine' or 'square':
- 'sine' is a sin wave that oscilates at 1/duration hz with an amplitude of depth.
- 'square' is a square wave that oscilates at 1/duration hz with an amplitude of depth.

##### duration:
Integer cycle length for oscillator in milliseconds.

##### depth:
The amount of pixel color shift that will be oscillated by the texture. Value is integer [1,255].

## Photosensitive Seizure Warning
###### (via Phillips Hue Documentation)
A very small percentage of people may experience a seizure when exposed to certain visual images, including flashing lights or patterns that may appear in video games. Even people who have no history of seizures or epilepsy may have an undiagnosed condition that can cause these “photosensitive epileptic seizures” while watching video with the additional light effects.These seizures may have a variety of symptoms, including lightheadedness, altered vision, eye or face twitching, jerking or shaking of arms or legs, disorientation, confusion, or momentary loss of awareness. Seizures may also cause loss of consciousness or convulsions that can lead to injury from falling down or striking nearby objects. Immediately stop project participation and consult a doctor if you experience any of these symptoms. Parents should watch for or ask their children about the above symptoms. Children and teenagers are more likely than adults to experience these seizures. The risk of photosensitive epileptic seizures may be reduced by taking the following precautions:

- Use it in a well-lit room
- Do not use it if you are drowsy or fatigued
- If you or any of your relatives have a history of seizures or epilepsy, consult a doctor before participation.
