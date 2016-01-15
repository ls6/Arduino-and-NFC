# Part 1

##Introduction

Here's an quite easy and not too expensive way to integrate NFC with Arduino.

Just in case, before we proceed, I'm going to make a disclaimer here. The NFC technology has been around for years begging to be adopted on a mass scale and when it have finally picked up the mess is considerable. For years the technology companies couldn't figure out the "killer app" and finally we now see there isn't one. Partly for this reason there's no consensus on the terminology. So my disclaimer is that I need to work with Mifare cards only so to keep the cost low I have selected the NFC modules from Stronglink which only talk to Mifare cards. And not even all of them. Nevertheless, Mifare enjoys the widest deployment in, I believe, the whole world besides Japan. Your public transport card most probably uses it. So, before you place your order double-check if this module covers whatever card you want to play with. I myself have purchased a set of NFC stickers from Stronglink to be able stick them on wherever needed.

##Hardware

An NFC (Mifare) module from Stronglink. I've chosen the SL030 because I wanted to work with 3.3V supply voltage but you can have a 5V version as well. Here you can see how big it is:

{% img /images/sl030.jpg %}
<small>As with most of casual bloggers you have to excuse the quality of pictures taken with a phone.</small>

The module comes as an board populated SMD components but _without headers_ so if you don't have spare ones laying around make sure you get them separately or you will have to solder wires directly to the board.

##Connecting the module

The board supports I<sup>2</sup>C interface so you need just 2 wires, power supply and ground to connect it to Arduino.

Arduino to SL030 wiring:
	A4/SDA 3
	A5/SCL 4
	GND    6
	3V3    1

Here's the board connected to Arduino Uno:
{% img ../../../../../../images/sl30Arduino.jpg %}

And to a [JeeNode](http://jeelabs.net/projects/hardware/wiki/JeeNode) (the reason I needed a 3.3V module is that this is the voltage the JeeNode operates on): 
{% img ../../../../../../images/sl30JeeNode.jpg %}

##Software

This NFC module is supported by the RFIDuino libraries hosted [here](http://github.com/marcboon/RFIDuino/) written by [Marc Boon](http://marcboon.com/) (thank you, Marc!). The SL030 module works with the library that Marc called SL018. On the screenshot below you can see both the simple example code (available in the "examples" subdirectory) as well as the result of scanning a number of tags.
<a href="../../../../../../images/JeeNodeOneReader.png" target="_blank">{% img ../../../../../../images/JeeNodeOneReader.png %}</a>

##What's next?

Well, that was pretty easy, wasn't it? I consider this part a warm-up and "check if everything is ok" step. My final goal is to [connect two readers](/blog/2012/02/27/arduino-and-nfc-part-2/) to a single Arduino and possibly connect it to internet. If you get bored before part 2 appears, take a look at Marc's code, there is a lot of comments that explain what is going on. You might also find the [SL030 User Manual](http://www.stronglink.cn/download/SL030-User-Manual.pdf) useful.

P.S. The JeeNode is a pretty cool device. You get an Arduino + Radio for less.

# Part 2

This is the continuation of the story of connecting unexpensive NFC readers to Arduino. [Part 1](/blog/2011/07/21/arduino-and-nfc-part-1/) explains how a Mifare reader from Stronglink can be easily connected. This time I'll show you how to connect more than one of them.

##Hardware configuration

The SL018 sports two jumpers which let you select the address the reader will use on an I<sup>2</sup>C bus. This means that you can connect up to four devices on a single bus and that before you connect more than one you have to solder one or more of these jumpers. Take look at the [manual](http://www.stronglink-rfid.com/download/SL030-User-Manual.pdf) for details.

##How is it going to work

[Marc's](http://marcboon.com/) [RFDuino](http://github.com/marcboon/RFIDuino/) can poll one reader at a time so we need a way to figure out when to poll which reader. Fortunately Stronglink's readers feature a pin that signals whether a card has been found in the field. On the SL030 this is pin number 5 called "Out", and we will connect it to one of the Arduino pins. Now, we will only poll the reader for card data once the reader have signalled that the card is actually available.

##Software

So, the implementation of the above method works roughly like this:
``` c
	//if the board has signalled that it found a tag
	if(!digitalRead(reader1OutPin)) 
	  {
	    //query tag data
	    rfid1.seekTag();
	    
	    //while we are waiting for data
	    while(!rfid1.available()) {
	    //break if the tag has been removed
	    if (digitalRead(reader1OutPin)) {
	      break;
	    };
	};
	tagString = rfid1.getTagString();
	Serial.println(tagString);
```		
We can add as many segments like this in the `loop()` part of the Arduino code and each segment supporting a particular reader will 'trigger' whenever that reader fill find a card.

The code where we are waiting for data could be written more naturally as:
``` c
	while(!digitalRead(reader1OutPin) || !rfid1.available());
```
but for some reason that I didn't have the time to investigte does not work.

##So, how to make it work exaclty?

Just go to Marc's [github repository](http://github.com/marcboon/RFIDuino/) and peek into the [RFIDuino/SL018/examples/twoReaders](https://github.com/marcboon/RFIDuino/tree/master/SL018/examples/twoReaders) directory. It contains a [twoReaders](https://github.com/marcboon/RFIDuino/blob/master/SL018/examples/twoReaders/twoReaders.ino) file that I have written and which Marc included in his repository. Please note that Marc updated his library for Arduino IDE 1.0 but there is nothing IDE-specific in this example. If you have an older version of the RFDUino library just change the file extension of my code accordingly.

Here's the full example code for your convenience:
``` c
    /**
     *  @title:  StrongLink SL018/SL030 RFID 2-reader demo
     *  @author: lukasz.szostek@gmail.com based on code of:
     *  @author: marc@marcboon.com
     *  @see:    http://www.stronglink.cn/english/sl018.htm
     *  @see:    http://www.stronglink.cn/english/sl030.htm
     *
     *  Arduino to SL018/SL030 wiring:
     *  A4/SDA     2     3
     *  A5/SCL     3     4
     *  5V         4     -
     *  GND        5     6
     *  3V3        -     1
     *  5          1     5 // first reader
     *  4          1     5 // second reader
     */

    #include <Wire.h>
    #include <SL018.h>
    
    //pins to listen for the RFID board signalling that it has detected a tag
    int reader1OutPin = 5;
    int reader2OutPin = 4;
    const char* tagString;
    
    SL018 rfid1;
    SL018 rfid2;
    
    void setup()
    {
      //make sure these two addresses match your reader configuration
      rfid1.address = 0x50;
      rfid2.address = 0x52;
      pinMode(reader1OutPin, INPUT);
      pinMode(reader2OutPin, INPUT);
      Wire.begin();
      Serial.begin(57600);
    
      // prompt for tag
      Serial.println("Show me your tag");
    }
    
    void loop()
    {
      //if the board has signalled that it found a tag
      if(!digitalRead(reader1OutPin)) 
      {
        //query tag data
    	rfid1.seekTag();
    	
    	/* for some reason the loop defined as:
        while(!digitalRead(reader1OutPin) || !rfid1.available());
    	does not work so we need a workaround:*/
    
	    //while we are waiting for data
	    while(!rfid1.available()) {
	      //break if the tag has been removed
          if (digitalRead(reader1OutPin)) {
            break;
          };
        };
        tagString = rfid1.getTagString();
        Serial.print("Reader 1 found: ");
        Serial.println(tagString);
	    //wait a while before querying the tag again
        delay(1500);
      };
      
      //"in parallel" we wait for the other reader
      if(!digitalRead(reader2OutPin)) 
      {
        rfid2.seekTag();
        while(!rfid2.available()) {
          if (digitalRead(reader2OutPin)) {
            break;
          };
        };
        tagString = rfid2.getTagString();
        Serial.print("Reader 2 found: ");
        Serial.println(tagString);
        delay(1500);  
      };
    }
```
Good luck and have fun! If you have some improvements to this code for [Marc's](https://github.com/marcboon/RFIDuino) (preferably) or [my](https://github.com/ls6/RFIDuino) repository on git hub and help pushing this library forward.
