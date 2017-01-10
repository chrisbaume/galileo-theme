---
layout: post
title: Beer monitoring with my Raspberry Pi
date: 2013-02-10 18:47:04.000000000 +00:00
type: post
published: true
comments: true
status: publish
categories:
- Raspberry Pi
tags: []
meta:
  _edit_last: '5947599'
  _publicize_pending: '1'
  _wpas_done_2804660: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:222300383;b:1;}}
  _wpas_done_2181874: '1'
  geo_public: '0'
  _oembed_7f923373df3be24559d0a0544082a825: "{{unknown}}"
  _oembed_ceca2a6795dc74a3d25f7db6b7e939a9: "{{unknown}}"
  _oembed_ea2c41b0a2acbb5d0ed8f50eb7e5c417: "{{unknown}}"
  _oembed_93e82aafff46db44a419a14e6e180575: "{{unknown}}"
  _oembed_576216acbe3810f3805e72e513d8f8a5: "{{unknown}}"
  _oembed_4fb63773093cb2fe53cb63c528a7a5d2: "{{unknown}}"
  _oembed_9151aa18bebca2de6f46d73047b325a6: "{{unknown}}"
  _oembed_d8036567f643d4f18c2b8a44ffe19cfb: "{{unknown}}"
  _oembed_14e59b0d22cceb0065c70f3779c03578: "{{unknown}}"
  _oembed_ed68cb4afde47ff1539df5e8f224ee15: "{{unknown}}"
  _oembed_9b3ac173dd6d3c2f103c41e82ff25704: "{{unknown}}"
author:
  login: chrisbaume
  email: chrisbaume@gmail.com
  display_name: chrisbaume
  first_name: ''
  last_name: ''
---
The secret to brewing great beer at home is making sure that you keep it at the right temperature. This can be tricky
when your house doesn’t have a thermostat and you’re not in the house for most of the day. However, by using a cheap
and cheerful sensor with a raspberry pi, you can record a log of the temperature and check it over the internet to make
sure your beer is brewing nicely.

## Hardware

The sensor I used is the DHT11 which, at the time of writing, you can order on eBay for £1.12 delivered. It has a
digital interface, so you don’t need to do any calibration or digital conversion as you would with a thermistor. To
connect the sensor to the RPi, all you need is a 10k resistor to pull-up the data signal and to make the following
connections (see pic).

* RPi VCC (pin 1) -> DHT11 pin 1
* RPi GPIO4 (pin 7) -> DHT11 pin 2
* RPi GND (pin 6) -> DHT11 pin 4

![Wiring diagram](/assets/dht11wiring.gif "Wiring diagram")

## Interfacing

The DHT11 uses its own serial interface, which can be interrogated using the wiringPi C library. To install and compile
the library, use the following commands:

{% highlight shell %}
sudo apt-get install git-core build-essential
git clone git://git.drogon.net/wiringPi
cd wiringPi
./build
{% endhighlight %}

[This blog post](http://www.rpiblog.com/2012/11/interfacing-temperature-and-humidity.html) details the code necessary
to read the sensor data. The code below has been modified to return only the current temperature and to repeat the
request if there is an error. Place the following into a file named dht11.c.

{% highlight cpp %}
#include <wiringPi.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#define MAX_TIME 85
#define DHT11PIN 7
#define ATTEMPTS 5
int dht11_val[5]={0,0,0,0,0};
 
int dht11_read_val()
{
  uint8_t lststate=HIGH;
  uint8_t counter=0;
  uint8_t j=0,i;
  for(i=0;i<5;i++)
     dht11_val[i]=0;
  pinMode(DHT11PIN,OUTPUT);
  digitalWrite(DHT11PIN,LOW);
  delay(18);
  digitalWrite(DHT11PIN,HIGH);
  delayMicroseconds(40);
  pinMode(DHT11PIN,INPUT);
  for(i=0;i<MAX_TIME;i++)
  {
    counter=0;
    while(digitalRead(DHT11PIN)==lststate){
      counter++;
      delayMicroseconds(1);
      if(counter==255)
        break;
    }
    lststate=digitalRead(DHT11PIN);
    if(counter==255)
       break;
    // top 3 transistions are ignored
    if((i>=4)&&(i%2==0)){
      dht11_val[j/8]<<=1;
      if(counter>16)
        dht11_val[j/8]|=1;
      j++;
    }
  }
  // verify checksum and print the verified data
  if((j>=40)&&(dht11_val[4]==((dht11_val[0]+dht11_val[1]+dht11_val[2]+dht11_val[3])& 0xFF)))
  {
    printf("%d.%d,%d.%d\n",dht11_val[0],dht11_val[1],dht11_val[2],dht11_val[3]);
    return 1;
  }
  else
    return 0;
}
 
int main(void)
{
  int attempts=ATTEMPTS;
  if(wiringPiSetup()==-1)
    exit(1);
  while(attempts)
  {
    int success = dht11_read_val();
    if (success) {
      break;
    }
    attempts--;
    delay(500);
  }
  return 0;
}
{% endhighlight %}

Compile and execute the source code with the following commands:

{% highlight shell %}
gcc -o dht11 dht11.c -L/usr/local/lib -lwiringPi -lpthread
sudo ./dht11
{% endhighlight %}

The program should return two numbers – one for relative humidity and the other for temperature.

## Logging

The simplest method of creating a log of the temp/humidity is to use a cronjob.  The program must be run as root, so
use the following command to edit the cron config file:

{% highlight shell %}
sudo crontab -e
{% endhighlight %}

Add the following line to the end. It will save a timestamp and the temp/humidity every minute to temp.log.

{% highlight shell %}
* * * * * echo `date +\%Y\%m\%d\%H\%M\%S`,`/home/pi/wiringPi/dht11` >> /home/pi/temp.log
{% endhighlight %}

If that doesn’t seem to be working, the system cron may not be running. The following command will start it, and the
line after will make sure it starts every time the RPi turns on:

{% highlight shell %}
sudo service cron start
sudo update-rc.d cron defaults
{% endhighlight %}

## Display

Reading a log file isn’t the easiest way to check how the temp/humidity is changing, but we can use a graphing library
to read the log file and plot it.  For this, I used [DyGraph](http://dygraphs.com/) which is a Javascript library for
plotting time-based information. Here is the final .html file I used:

{% highlight html %}
<html>
<head>
<script type="text/javascript" src="dygraph-combined.js"></script>
</head>
<body>
<div id="graphdiv" style="width:750px; height:400px;"></div>
<script type="text/javascript">
 
  function addZero(num)
  {
    var s=num+"";
    if (s.length < 2) s="0"+s;
    return s;
  }
 
  function dateFormat(indate)
  {
    var hh = addZero(indate.getHours());
    var MM = addZero(indate.getMinutes());
    //var ss = addZero(indate.getSeconds());
    var dd = addZero(indate.getDate());
    var mm = addZero(indate.getMonth()+1);
    var yyyy = addZero(indate.getFullYear());
    return dd+'/'+mm+' '+hh+':'+MM;
  }
 
  g = new Dygraph(
    document.getElementById("graphdiv"),
    "temp.log",
    {
      xValueParser: function(x) {
        var date = new Date(x.replace(
          /^(\d{4})(\d\d)(\d\d)(\d\d)(\d\d)(\d\d)$/,
          '$4:$5:$6 $2/$3/$1'
        ));
        return date.getTime();
      },
      axes: {
        x: {
          ticker: Dygraph.dateTicker,
          axisLabelFormatter: function(x) {
            return dateFormat(new Date(x));
          },
          valueFormatter: function(x) {
            return dateFormat(new Date(x));
          }
        }
      },
      labelsDivWidth: 310,
      rollPeriod: 30,
      strokeWidth: 2.0,
      labels: ['Date','Humidity (%)','Temp (&deg;C)']
    }
  );
</script>
</body>
</html>
{% endhighlight %}

You can set this up on the RPi by installing a web server like Apache, creating a symlink to the log file, downloading
dygraph and saving the above html file as /var/www/temp.html.

{% highlight shell %}
sudo apt-get install apache2
ln -s /home/pi/temp.log /var/www/temp.log
wget -P /var/www http://dygraphs.com/dygraph-combined.js
{% endhighlight %}

Then use your browser to navigate to 192.168.1.2/temp.html, replacing the IP address with that of your RPi. To be able
to access the page from the internet, you’ll have to set up NAT on your router.

Here’s what my final system looks like:

![Temperature graph](/assets/temp.png "Temperature graph")
