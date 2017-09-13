# High Frequency Data framework, meant for WiFi-based robots

## What it does, and why
 - Provides helpers that deal with the entire communications protocol in as few as 6 lines of code!
 - Is a Java-native alternative to Network Tables for FRC robots not on a live field - no more dealing with missing runtime libraries just to receive data in the lab!
 - Seamlessly integrates with Network Tables for live FRC field environments (including NT's protections & reliability)
 - Deals with threading & socket comms internally, so robot code doesn't have to.
 - Is available for UDP, TCP, NetworkTables, and 'passthrough' protocols
 - Automatically compresses null data, in order to encourage the best practice of not reading sensors when they aren't needed
 - Auto-increments metadata of cycle id & relative nanosecond timestamp for Codexes
 - Includes benchmark & data integrity tests to ensure encode/decode/transit processing times are minimized

## Getting started
Get the artifact!
```
groupId: com.flybotix
artifactId: HighFrequencyRobots
version: 0.0.1
```

1. Create an enumeration that describes your data while implementing the CodexOf interface.  Both sides of the comms link will need this enumeration at compile time.  Since enumerations are lists of static objects, you can also do anything else you want with your enumeration (descriptions, short descriptions, inheritance, etc).
```java
import com.flybotix.hfr.codex.CodexOf;
public enum RobotData implements CodexOf<Double>{
  pdb0,
  pdb1,
  pdb2,
  pdb3,
  vrm0,
  totalcurrent,
  gyro
}
```
2. On your robot, create a "sender".
```java
CodexSender<Double, RobotData> sender = new CodexSender<>(RobotData.class, true);
sender.initConnection(EProtocol.UDP, 7778, 7777, "localhost");
```
3. On the laptop/client side, create a 'receiver'.  Then register for updates with that receiver.
```java
CodexReceiver<Double, RobotData> receiver = new CodexReceiver<>(RobotData.class);
receiver.startReceiving(EProtocol.UDP, 7777, "");
receiver.addListener(codex -> System.out.println(codex));
```
4.  During robot initialization, create a Codex and pass its reference where it's needed.  Try not to create a 'new' Codex for the same enumeration, as that may cause Java's garbage collection to pause the robot.
```java
// During robot init
Codex<Double, RobotData> data = Codex.of.thisEnum(RobotData.class);

// Reset the codex at the beginning of each cycle.  This effectively sets each value to 'null'.  Fill out data throughout each cycle.
data.reset(); // beginning of the cycle
data.put(RobotData.pdb2, -23.3d);
data.put(RobotData.gyro, mxp.getGyroRelative());

// Use the data throughout the robot cycle after it's gathered
double degrees = data.get(RobotData.gyro);

// Send the data back to the laptop at the end of each cycle
sender.send(data);
```

## The Codex: an enumerated array
(WIP)
This project is based upon two simple principles:
1. A piece of data in an array actually has two pieces of information: the data value, and the position of the data in the array.
2. We'd rather see compiler errors instead of weird data when the robot is running.  Compiler errors are easy to debug and fix.  Weird data is not.

Enumerations in Java are just that: a compile-time reference of objects in a static array.  To get the position of the enumeration in the array, we call enum.ordinal().  To get the static information of the enumeration, we can call name(), toString(), or any other implemented method.  Enumerations can even implement interfaces, making them (effectively) static lambdas.  The HFR project expects a Codex's enumeration to be available at compile time on all "sides" of a comms link.

The Codex in this project is an array that uses an enumeration to define what the values at any given position MEAN to a programmer & user. A programmer simply needs to specify the type in the array, and then the enumeration which backs the codex.  The class has a few utility and metadata methods.

An enumerated array was chosen over a EnumMap because indexing an array is a faster O(1) operation, whereas indexing into a map is a O(n) operation.  Since an enumeration's length never changes after compile-time, we know that the length of the data array will never change - and thus most operations into getting data from the array are safe.

The fact that a Codex must represent data of the same type is simultaneously this project's biggest advantage and disadvantage.  It means this project will never represent complex types (e.g. like what JSON can do), but it also means that communicating the data can be extremely efficient (unlike JSON) - and therefore be executed at a higher frequency.  In FRC robots (and many IoT scenarios), the data is all of the same type.  If the data isn't sent over a comms protocol, then the type of the codex _can_ be a String, array of arrays, complex POJOs, etc, without worry of data corruption.
