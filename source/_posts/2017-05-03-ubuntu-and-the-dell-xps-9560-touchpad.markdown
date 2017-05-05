---
published: true
layout: post
title: "Ubuntu and the Dell XPS 9560 Touchpad"
date: 2017-05-03 23:19:47 +0000
comments: true
categories: 
---

I recently purchased a [Dell XPS 15 9560][xps_15] which I use for my day to day work, with Ubuntu 16.04 as the only operating system. It is a wonderful laptop and Ubuntu runs on it with virtually no issues. There is one issue which bothered me more than others. The touch pad sensitivity and palm detection. Whenever I tried to type a long piece of text I would invariably find my self type a couple of words somewhere else on the screen. 

The problem was that when I reached for letters in the middle of the keyboard such as `T` or `Y` the inside bit of my palm would touch the touchpad in the top corner. The touchpad percieved that as a tap and I'd continue typing wherever the mouse pointer was at the time.

I usually have a mouse plugged in and have set the touchpad to be disabled when a mouse is plugged in. I used [these instructions][disable_tp_on_mouse] from askubuntu to do that. It helped but I do quite a bit of work away from a desk and then I need the touchpad enabled. 

After a bit of googling a found a couple of questions on answers on numerous forums and managed to piece things together from there. *See the references at the bottom of the post.* 

To make it work you first you have work out what your touchpad is called internally. You do this by running:

```
    xinput --list
```
which prints something like this:

{% img center /images/ubuntu_dell_touchpad/x-input-list.png 400 'x-input --list' %}

From this output you can see there are a number of touch input devices:

* Virtual core XTEST pointer - __Not entirely sure what this is__
* ELAN Touchscreen
* DLL07BE:01 06CB:7A13 Touchpad 

Look for the one called 'Touchpad'. You can now list all the configuration for this device by using `xinput list-props`. For the XPS it is:

```
    xinput list-props  "DLL07BE:01 06CB:7A13 Touchpad"
```

This yields a long list of properties and their values. Look for the following:

* Synaptics Palm Detection
* Synaptics Palm Dimensions
* Synaptics Area

In my case these where set as follows:

| Property                  | Value       | Description                                 |
|---------------------------|-------------|---------------------------------------------|
| Synaptics Palm Detection  | 0           | This means palm detection is off            |
| Synaptics Palm Dimensions | 10, 200     | The first value refer to the minimum        |
| Synaptics Area            | 0, 0, 0, 0  | This describe the touchpad surface area where touches are detected. 

<br/>
Most of the references I found was talking about changing the first two properties `Synaptics Palm Detection` and `Synaptics Palm Dimensions` however, changing those 
didn't make a difference for me. The cursor still jump around because my palm looks like a finger at the edge of the touchpad no matter how small I make the palm detection setting. This is understandable since only a small part of my palm actually touches the touchpad surface while typing.

The setting which made the biggest difference for me was the last one `Synaptics Area`. It is used to manage the detection of touches at the edges of the touchpad. 
By changing the four values associated with Synaptics Area you can change the area of the touchpad that is active to touches. 

> <small> Note that the `Synaptics Area` is about inital touches. The disabled areas still work if a touch is initated in the active area</small>

The first value defines how far from the left of the touchpad edge touches are detected. Anything to the left of this value is not considered a touch. The second value 
sets how far to the right the active part of the touchpad stretches. The third sets how far from the top edge the active area starts and the fourth is how far down the 
active part stretches.

To configure these you first have to work out how large the touchpad is by running the following command:

```
    less /var/log/Xorg.0.log | grep -i range
```

```
    [     6.904] (--) synaptics: DLL07BE:01 06CB:7A13 Touchpad: x-axis range 0 - 1228 (res 12)
    [     6.904] (--) synaptics: DLL07BE:01 06CB:7A13 Touchpad: y-axis range 0 - 928 (res 12)
    [     6.904] (--) synaptics: DLL07BE:01 06CB:7A13 Touchpad: invalid pressure range.  defaulting to 0 - 255
    [     6.904] (--) synaptics: DLL07BE:01 06CB:7A13 Touchpad: invalid finger width range.  defaulting to 0 - 15
    [     7.028] (--) synaptics: SynPS/2 Synaptics TouchPad: x-axis range 1278 - 5664 (res 0)
    [     7.028] (--) synaptics: SynPS/2 Synaptics TouchPad: y-axis range 1206 - 4646 (res 0)
    [     7.028] (--) synaptics: SynPS/2 Synaptics TouchPad: pressure range 0 - 255
    [     7.028] (--) synaptics: SynPS/2 Synaptics TouchPad: finger width range 0 - 15
```

From this you can see that the touchpad as a horizontal range of `0-1228` and a vertical range of `0-928`. I don't know what these numbers mean or measure, but I 
played around with different values a bit and found that, for me, the magic number is `70`. 

```
xinput set-prop "DLL07BE:01 06CB:7A13 Touchpad" "Synaptics Area" 70 1168 70 0
```
* Start detecting touches `70` from the left
* Stop detecting touches `1228-70 = 1168` from the left
* Start detecing touches `70` from the top
* Detect touches all the way down

This setup work perfectly for me without even changing the `Synaptics Palm Dimensions`. I can now type without worrying about my cursor jumping all over the place. The best part is that if you initiate a drag of the pointer in the active area of the touchpad, the touchpad will track your finger all the way to the edge, even in the 'no touch' zone.

To make the changes permanent put them in a script and run the file at log in time using the [startup applications](https://www.howtogeek.com/189995/how-to-manage-startup-applications-in-ubuntu-14.04/) gui. 

__References:__

* [Fixing palm detect on Ubuntu 14.04](https://stevenkohlmeyer.com/fixing-palm-detect-ubuntu-14-04/)
* [Fixing palm detect on Ubuntu 14.04](https://stevenkohlmeyer.com/fixing-palm-detect-ubuntu-14-04/)
* [Tune touchpad for smaller area](https://askubuntu.com/questions/221664/how-to-tune-touchpad-for-smaller-area)
* [Synaptics Area](ftp://www.x.org/pub/X11R7.5/doc/man/man4/synaptics.4.html)

[xps_15]: http://www.dell.com/au/p/xps-15-9560-laptop/pd?ref=PD_OC
[disable_tp_on_mouse]: https://askubuntu.com/questions/787433/how-do-i-disable-touchpad-when-using-a-mouse