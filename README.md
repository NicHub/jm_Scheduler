<!--

2016-07-14: In progress...
2016-07-08: Initial commit.

Licence: LGPL v3

Author: Jean-Marc Paratte
Email: jean-marc@paratte.ch

-->

<img src="http://jean-marc.paratte.ch/wp-content/uploads/2013/01/diduino1_960x96.jpg" class="header-image" alt="jmP" height="96" width="960">

# jm_Scheduler - A Scheduler Library for Arduino

2016-07-08: Initial commit

### Concept

**jm_Scheduler** schedules repeated and intervaled routines like the JavaScript `setInterval()` function does,
but with some improvements:

- By default, **jm_Scheduler** starts immediately the routine and repeats it periodically.
- The first execution can be differed.
- The repeated executions can be voided.
- The interval between executions can be dynamically modified.
- The execution can be stopped and later restarted.
- The executed routine can be dynamically changed.

**jm_Scheduler** doesn't schedule like the official [**Scheduler** Library for Arduino DUE and ZERO](https://www.arduino.cc/en/Reference/Scheduler) does,
`yield()` function which suspends the task is not implemented,
`startLoop()` function which creates a new _stack_ for the task is not implemented.

**jm_Scheduler** schedules tasks sequentially on the stack processor.
The rules to _yield_ and _resume_ are:

- _yield_ comes out when _routine_ leaves at end of function or by an explicit `return` instruction.
- _resume_ to a next _state_ can be done with a variable and a _switch_ instruction. Or:
- _resume_ to a next _state_ can be done by switching to another function.
- Persistent variables must be implemented _globally_.

### Basic Example

	// This example schedules a routine every second
	
	#include <jm_Scheduler.h>
  
	jm_Scheduler scheduler;
	
	void routine()
	{
		Serial.print('.');
	}
  
	void setup(void)
	{
		Serial.begin(9600);
		
		scheduler.start(routine, TIMESTAMP_1SEC); // Start immediately routine() and repeat it every second
	}
  
	void loop(void)
	{
		jm_Scheduler::cycle();
	}

### Study Plan

- Begin with example **Clock1.ino**. This example demonstrates the advantages to start immediately a time display routine and periodically repeat it.
- Follow with examples **Clock2.ino** and **Clock3.ino** which present other timing ways.
- **Clock4.ino** example presents a usefully **jm_Scheduler** technical: changing dynamically the routine to execute.
- **Beat1.ino** and **Beat2.ino** examples present interaction between 2 scheduling routines.
- **Wakeup1.ino** example demonstrates the possible interaction between an interrupt and a scheduled routine, implementing a timeout.

### Timestamp

The _timestamp_ is read from the Arduino function `micros()`.
By design, the `micros()` function of Arduino UNO and Leonardo running at 16MHz returns a _[us]_ _timestamp_ with a resolution of _[4us]_.

`micros()` declaration is:

```C
unsigned long micros()
```

Look at https://www.arduino.cc/en/Reference/Micros for details.

<!--
### More about Timestamp
-->

_timestamp_ is a 32bit _[us]_ counter and it overflows about every 70 minutes (precisely 1h+11m+34s+967ms+296us).

<!--
The periodicity of 70 minutes is sometimes not enough to control slow processes.
Look next section for answers and tricks.
-->

### Timestamp declaration and constants

```C
typedef uint32_t timestamp_t;

#define timestamp_read() ((timestamp_t)micros())

#define TIMESTAMP_DEAD (0x01CA0000) // routine dead time [30s + 15ms + 488us]
#define TIMESTAMP_TMAX (0xFE35FFFF) // [1h + 11m + 4s + 951ms + 808us - 1]

#define TIMESTAMP_1US	(1UL)					// [1us]
#define TIMESTAMP_1MS	(1000*TIMESTAMP_1US)	// [1ms]
#define TIMESTAMP_1SEC	(1000*TIMESTAMP_1MS)	// [1s]
#define TIMESTAMP_1MIN	(60*TIMESTAMP_1SEC)		// [1 minute]
#define TIMESTAMP_1HOUR	(60*TIMESTAMP_1MIN)		// [1 hour]
```

> `timestamp_t` defines the type of all _timestamp_ values.

> `timestamp_read()` returns the instantaneous _timestamp_.
This function can also be used by interrupt routines to _timestamp_ they data.

> `TIMESTAMP_DEAD` is the maximum allowed execution time of a routine to guarantee right scheduling.
If the routine doesn't end before, the scheduler could miss very long scheduling (see next).

> `TIMESTAMP_TMAX` is the maximum allowed scheduling time of a routine.
In practice, don't use _timestamp_ values greater than 1 hour.

### jm_Scheduler functions

```C
// start routine immediately
void start(voidfuncptr_t func);

// start routine immediately and repeat it at fixed interval
void start(voidfuncptr_t func, timestamp_t ival);

// start routine on time and repeat it at fixed interval
void start(voidfuncptr_t func, timestamp_t time, timestamp_t ival);

// stop routine, current or scheduled, remove it from chain
void stop();

// rearm current routine and set or reset interval
void rearm(timestamp_t ival);

// rearm current routine, change routine function and set or reset interval
void rearm(voidfuncptr_t func, timestamp_t ival);
```

> `start()` starts a routine immediately or on time, with or without repetitions.
`start()` is invoked one time for a given scheduler variable.

> `stop()` cancels further execution of scheduled task. 
`stop()` can be invoked from inside _routine_ or from other program parts.
If invoked from inside _routine_, `stop()` doesn't exit the function, just cancels further execution.

> `rearm()` changes the values of the given scheduler variable.
The new values are evaluated on exit _routine_ function.
The main usage is to change _interval_ or change _function_ or else cancel further execution.

### Good scheduling practices

- To guarantee a good scheduling of all managed tasks,
the execution time of each function must be as short as possible.
- Avoid `delay()` function, replace with `rearm()` function and appropriate arguments.

### Changing of Timestamp

Here are some hacks that can be implemented by modifying the file **jm_Scheduler.h**.

- Another choice for the _timestamp_ resolution could be the _[ms]_ read from the Arduino function `millis()`. 
- Gain speed during _timestamp_ comparison by shortening the size to 16bit.
- Obtain very long periodicity by implementing a 64bit _timestamp_.
