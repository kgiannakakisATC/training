Complex Event Processing
===

> **Last updated:** August 24th 2015

# Tutorial - A heating control system

To describe all the features avaible on the end of august, we are going to take an example: an heating control system.

## Running example
Marc supervises the electric consumption of a building aiming at reducing the electric consumption. His first step to achieve this goal is to implement alerts when heating is on in a room while the window is open.

To be accepted by the inhabitants of the building, the alert system should be flexible enough. For example, an alert should not be triggered as soon as the window is opened while the heating is on, to allow inhabitants to turn the heating off on their own shortly after, without a reminder, which could be perceived negatively. Also, as the building offers limited automation (windows cannot be automatically opened/closed and heating cannot be automatically turned on/off), Marc wants to be able to detect the presence of a person in a room using the available sensors (light, temperature, humidity and a sensor detecting whether heating is on or off), which do not include a motion sensor. This way, alerts can be triggered when the person is in the room so that the person can close the window or turn off the heating.

As Marc is not allowed to buy extra sensors, he should rely on the sensors available in each room, in addition to a global temperature sensor installed on the roof, to implement his alert system. The final system is depicted in the following figure.

![heating system](https://github.com/HEADS-project/training/blob/master/1.ThingML_Basics/6.CEP/docs/heatingSystem.png)

On the figure, all the blue elements are virtual sensors whereas the others are physical sensors, except the two hands in the bottom. These elements represent the alert system. The physical sensors send the value regularly. The virtual sensors are micro-controllers which receive values from other sensors, physical or virtual, and apply the rules defined by an user. On the center, the virtual sensor is a micro-controller which turns on the alert system, shown by the hand with a green card, or turn off, shown by the hand with a red card.

On the left of the figure, there is an indoor temperature sensor, a humidity sensor and a light sensor. All these sensors send a value to a micro-controller, shown by the person with a question mark on the figure. This device behaves as a person sensor : based on the values send by the three sensors, it can compute a boolean which is true when a person is in the room, false otherwise. To compute these values, it applies on the last values from each sensor an algorithm to get a temporary values. After, it analyzes the evolution of this value during a time windows. And, for example, if the mean of the temporary values are greater than a threshold during 30s, so the sensor sends a true boolean, which mean that a person is in the piece.

On the top of the figure, we have the heating sensor which sends regularly its values, without any modification. The value is just a boolean which is true when the heating is on, false otherwise.

On the right of the figure, there is an indoor temperature and an outdoor temperature. The sensors values are sent to the virtual sensor shown by a blue window, which is also a micro-controller. Based on this two values, the device can detect if a window is open or not. To do this, it compares the last value of both temperature sensors. Then, it sends a true boolean if a window is open, false otherwise.

On the middle, we have a virtual sensor which combines the values from the person sensor, the heating sensor and the window sensor. This virtual sensor combines the last values of each source sensors. If, in a room, there is a person and the window is open and the heating is on during at least a predefined time (e.g five minutes), then the micro-controller turn on the alert system, shown by the red card.

As we can see, the micro-controllers combine values received from different sensors, with sometimes the time window concept. There is no easy way to implement that with just state machine. Indeed, the developers must implement time window manually, use temporary values, use booleans to combine values. However, it already exists some technique to do this : complex event processing (CEP). To ease such development, in the HEADS project we are implementing a new approach which combine state machine and CEP, with resources constraints (device with a few KB of RAM and a few MHz of clock speed). We describe this method in the next section.

## How to implement this in ThingML
All the code of this example can be find here.

### The heating sensor - on the top
To declare a stream, two things should be specified: the input event (or source) and the output event. In ThingML, there is two keywords: **from** to declare the source and **action** to declare the output. Both must be in a **stream** declaration, in the same level as function declation : 
```
stream heatingSensor do
    from rcvPort?heatingIsOn
    action sendPort!cepHeatingStatus()
end
```

This piece of code allows to send a "cepHeatingStatus" when a "heatingIsOn" message is received. The language permits also to add some values to the output event (and get the values of the sources) with the **select** clause. **ATTENTION:** all the output parameters must be in the **select** clause, even if it is a literal. Thus, the below code shows how to forward the value of a message in ThingML:

```
stream heatingSensor do
    from heating : [rcvPort?heatingIsOn]
    select v : heating.value
    action sendPort!cepHeatingStatus(v)
end
```

Any expression (complex arithmetic expression, function call, ...) can be added in this clause. If there is more than once parameter, the declarations are separated by a coma. This clause is executed just before the action. In following paragraphs, we will see some operators. Thus, the select clause occurs after all of them. 


### The windows-open detector (on the right) and the person-presence detector (on the left)
Currently, two events can be joined to create a unique output. More than declare this two messages, the result message with its new parameters must be specified (on the below example, all the code after the arrow). Any expression can be added to compute the new parameter. For example, here we put a function call. 

```
stream windowOpenDetector do
   from w : [ indoor : rcvPort?indoorTemp & outdoor : rcvPort?outdoorTemp -> cepWindowOpen(complexAlgo(indoor.value,outdoor.value))]
   select windowIsOpen: w.v
   action sendPort!cepWindowOpen(windowIsOpen) 
end
```

However, currently we can join only two events. So, for the person-presence detector, two streams have to be created, like in the below code:

```
stream joinTempHumidity do
   from jTH : [ temp: rcvPort?indoorTemp & humi: rcvPort?humidity -> cepJTH(temp.v,humi.v) ]
   select tempV : jTH.temp, humiV : jTH.humi
   action sendPort!cepJTH(tempV,humiV)
end

stream personPresenseDetector do
   from person : [ joinTH: rcvPort?cepJTH & light : rcvPort?light -> cepPersonIsPresent(complexAlgo(joinTH.temp,joinTH.humi,light.value)) ]
   select isPresent: person.v
   action sendPort!cepPersonIsPresent(isPresent)
end
```

**ATTENTION:** the semantic of the join is the one of [ReactiveX](http://reactivex.io/documentation/operators/join.html). 
It is not possible (currently) to customize the time in ThingML. Currently, you have to modify the generated source code. 

### Merge multiple window-open detector
We can imagine that there is a window-open detecting device on each window. But, in our example, if one window is open, it doesn't matter about which, we notify the system. To develop this, the ThingML language support the **merge** feature. The output event is sent as soon as one event of the source list is received. Contrary to the join feature, the merge feature support an "infinite" number of merged messages. **ATTENTION:** However, all the messages must have the same footprint (i.e: same parameters type).

Since the message parameters could have different name, we introduce a new syntax element: "#i", where i is the index of the parameter (we start to count at 0). The following code shows how to merge three window-open detectors:

```
stream mergeWindowOpenDetector do
    from merge: [ window1?cepWindowOpen | window2?cepWindowOpen | window3?cepWindowOpen -> cepWindowOpen(#0)]
    select oneWindowOpen: merge.v
    action room1!cepWindowOpen(oneWindowOpen)
end
```

This syntax can be also used in the **select** clause. For example, here the select could be : `select oneWindowOpen: #0 `.

### Operators
In all the previous code, the output event is sent whatever is value. However, we can filter these messages to send only message that interested in the system. It can be done thanks to the **filter** operator. It needs one thing: a special "function" which takes a message as parameter and return a boolean. We talk about **operator**, and you can declare one as following:
```
operator filterHeatingValue(msg : heatingIsOn) : Boolean do
    return msg.value //a message is sent only if the value is true
end
```

Then, we modify the stream declaration to call this filter:
```
stream heatingSensor do
    from heating : [rcvPort?heatingIsOn]::filter(filterHeatingValue(heating))
    select v : heating.value
    action sendPort!cepHeatingStatus(v)
end
```

Now, you can modify all the streams that need to be modified so that sends only a message with a true value. You can add how many filter you want, and where you want. It means that you can filter the different sources of a merge or a join. For example, with the previous merge example, we could get (this is probably not the best solution):
```
operator filterWindowValue(m : cepWindowOpen) : Boolean do
    return m.v
end

stream mergeWindowOpenDetector do
    from merge: [ w1 : [window1?cepWindowOpen]::filter(filterWindowValue(w1)) | w2 : [window2?cepWindowOpen]::filter(filterWindowValue(w2)) | w3 : [window3?cepWindowOpen]::filter(filterWindowValue(w3)) -> cepWindowOpen(#0)]
    select oneWindowOpen: merge.v
    action room1!cepWindowOpen(oneWindowOpen)
end
```

Another feature is to subdivise a stream to process just a part, we talk about window. We have two kinds of window: length and time window. Both allow the developer to do some aggregation operation (count, average, max, min) on a part of the streams. To create a length window, the size should be declared but the step is optional. If the step is not specified, its value is equal to the size. A new window is created every step items and contains at most size events. For the time window, instead of declare the size/step with number of events, there are declared with time, in milliseconds. **ATTENTION:** currently it is not possible to apply an aggregator function on a message which does not have any parameter. And, you can manipulate the collection of messages only in the stream declaration. You cannot send it as a message parameter.

A version of all basic function (sum, average, max, min and count) are available [here](https://github.com/SINTEF-9012/ThingML/tree/master/org.thingml.samples/src/main/thingml/cep). 

For example, we can compute the average of values in a time window, as below: 
```
stream averageValues do
    from ints : [recvPort?intValue]::timeWindow(50,10)
    select avg: avg(ints.value[])
    action sendPort!avgValues(avg)
end
```

The keyword for a length window is **lengthWindow**. **ATTENTION:** even if it's possible, the window operators must be the last operators and cannot be chained. I do not assure the result. It should be a next feature.

Now, you have all the keys to implement the Marc's system. To simulate the physical sensor, we can use the following files:
* `LightSensor.thingml`,
* `heatingSensor.thingml`,
* `humiditySensor.thingml`,
* `indoorTemperatureSensor.thingml`,
* `outdoorTemperatureSensor.thingml`.


## Your turn to play!
In this last step of the tutorial, your task is to write your own program in ThingML.

**Your program should contains:**
* at least one simple stream and one "complex stream" (with merge or join)
* at least a stream with a filter operator
* at least a stream with a window (time or length) operator

Submit your program by forking the training repository, adding to example in a sub-folder in the [Contribs](https://github.com/HEADS-project/training/tree/master/1.ThingML_Basics/5.Contrib) directory and making pull request. If you have no idea how to do this, send your example to Brice Morin (brice.morin@sintef.no). The best examples will be added as additional examples in this tutorial. Think about what examples you would have liked to get!
