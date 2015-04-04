RunningAverage
================================================

RunningAverage keeps a running average of your sampled data without blowing up your memory 

- The library stores N individual values in a circular buffer to calculate the running average.
- This code has been modified to work on the Spark Core by Lucas Choate

[See the original Arduino Code:](http://playground.arduino.cc/Main/RunningAverage)

One of the main applications for the Arduino board is reading and logging of sensor data. For instance one monitors pressure every second of the day. As high sample rates often generates "spikes" in the graphs one also wants to have an average of the measurements. As the measurements are not static in time what we often need is a running average. This is the average of a certain period and very valuable when doing trend analysis.

Simplest form of a running average can be done by code that builds upon the "previous" running average:

```Arduino
float alpha = 0.7; // factor to tune
value = alpha * measurement + (1-alpha) * value;
```

If one doesn't want to use floating point math - as this takes up memory and decreases speed - one can do the same completely in the integer domain. The division by 256 in the sample code is a shift-right 8, which is faster than say division by e.g. 100. This is true for every power of 2 as divider and one only must take care the sum of the weigths equals the power of 2. And of course one should take care there is no intermediate overflow (consider using unsigned long)

```Arduino
#define POWER 256
int alpha = 178;
value = (alpha * measurement + (POWER - alpha) * value )/ POWER;
```

If you need a more accurate running average, in concreto from the last 10 measurements, you need an array (or linked list) to hold them. This array acts as a circular buffer and with every new measurement the oldest one is removed. The running average is calculated as the sum of all elements divided by the number of elements in the array. The code for the running average will be something like this:

```Arduino
long runningAverage(int M)
{
  static int LM[10];      // LastMeasurements
  static byte index = 0;
  static long sum = 0;
  static byte count = 0;

  // keep sum updated to improve speed.
  sum -= LM[index];
  LM[index] = M;
  sum += LM[index];
  index = index % LMSIZE;
  if (count < LMSIZE) count++;

  return sum / count;
}
```

Drawback of this code is that the array to hold all values can become quite large. If you have one measurement per second and you want a running average per minute you need an array of 60; an average per hour would need an array of 3600!. That couldn't be done this way on an Arduino as it only has 2K of RAM. However by building a 2 stage average it can be approached quite well (disclaimer: not for all measurements). In psuedo code:

```
every second:   rapm = runningAverageMinute(measurement);
every minute:   raph = runningAverageHour(rapm);
```

As a new internal static array is needed for every runningAverage function, this screams to be implemented as a class.

#RunningAverage library

The runningAverage library makes a class of the function above so it can be used multiple times in an sketch. It decouples the add() and the avg() function to be a bit more flexible e.g. one can call the average multiple times without adding a thing. Please note that every instance of the class adds its own array to hold measurements, and that this adds up to the memory usage. The interface of the class is kept as small as possible.

##Note: with version 0.2 the names of the methods are all made more descriptive.

```Arduino
  RunningAverage(int);		// constructor; int=size of internal array;
  ~RunningAverage();		// destructor;  
  void clear();			// reset all counters
  void addValue(float);		// add a value (and remove an old one)
  float getAverage();		// get the running average
  void fillValue(float, int);	// fill with n x value

// backwards compatibility
// clr() clear()
// add(x) addValue(x)
// avg() getAverage()

    // new in 0.2.04 version
    float getElement(uint8_t idx);
    uint8_t getSize() { return _size; }
    uint8_t getCount() { return _cnt; }
```

#Usage

A small sketch shows how it can be used. A random generator is used to mimic a sensor.

```Arduino
//
//    FILE: runningAverageTest.pde
//  AUTHOR: Rob Tillaart
//    DATE: 2012-12-30
//
// PUPROSE: show working of runningAverage
//

#include "RunningAverage.h"

RunningAverage myRA(10);
int samples = 0;

void setup(void) 
{
  Serial.begin(115200);
  Serial.println("Demo RunningAverage lib");
  Serial.print("Version: ");
  Serial.println(RUNNINGAVERAGE_LIB_VERSION);
  myRA.clear(); // explicitly start clean
}

void loop(void) 
{
  long rn = random(0, 1000);
  myRA.addValue(rn * 0.001);
  samples++;
  Serial.print("Running Average: ");
  Serial.println(myRA.getAverage(), 3);

  if (samples == 300)
  {
    samples = 0;
    myRA.clear();
  }
  delay(100);
}
```

In setup() the myRA is cleared so we can start adding new data.

In loop() first a random number is generated and converted to a float to be added to myRA. Then the runningAverage is printed to the serial port. One could also display it on some LCD or send over ethernet etc. When 300 items are added myRA is cleared to start over again.

##Notes

To use the library, make a folder in your SKETCHBOOKPATH\libaries with the name RunningAverage and put the .h and .cpp there. Optionally make a examples subdirectory to place the sample app.

##History

2011-01-30: initial version
2011-02-28: fixed missing destructor in .h file
2011-02-28: removed default constructor;
2012-??-??: trimValue()Yuval Naveh added trimValue (found on web)
2012-11-21: refactored
2012-12-30: added fillValue() + refactored for publishing
2014-07-03: added memory protection code - if internal array cannot be allocated size becomes 0. This is to solve issue described here - http://forum.arduino.cc/index.php?topic=50473.msg1790086#msg1790086 -

##Todo

Test extensively ...
Template class
Enjoy tinkering,
rob.tillaart@removethisgmail.com