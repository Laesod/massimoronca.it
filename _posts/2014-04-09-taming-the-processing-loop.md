---
layout: post
title: Taming the processing loop
excerpt: A simple use case for state machines in Processing
source-name: MIKAMAYHEM
source-url: http://dev.mikamai.com/post/82178345433/taming-the-processing-loop
---

In Mikamai we do a lot of reasearch on [non](http://dev.mikamai.com/post/78652180658/how-to-program-an-attiny85-or-attiny45-with-an) [conventional](http://dev.mikamai.com/post/78453410376/let-your-raspberry-pi-see-this-wonderful-world) [hardware](http://dev.mikamai.com/post/69163914657/intel-galileo-getting-started-with-mac-os-x), we make [prototypes](http://dev.mikamai.com/post/76945627390/you-cant-touch-this-an-evil-arduino-based-alarm) or create unsual interfaces that are very domain specific.  

Like this one 

![image](https://scontent-b-ams.xx.fbcdn.net/hphotos-ash3/t1.0-9/994503_10151525258526336_667825845_n.jpg)  

Seriously, we did it.  

To quickly sketch ideas, we often rely on [Processing](http://www.processing.org/), it's super easy and its loop based execution model gives the feeling of programming a video game.  
The drawback is that it is so fast to get something working, that you will be tempted to make the mistake of creating a [polished prototpe](http://foxdellfolio.com/the-perils-of-a-polished-prototype/).  
Your prototype code ends up in production and there's no way back from there.  

To resist the temptation of releasing a blob of code, I borrowed [a technique from  one of the Rob Pike's talks](https://www.youtube.com/watch?v=HxaD_trXwRE) to keep things easy, while keeping them clean at the same time.    

It is basically an implementation of a state machime.  
We're gonna have a `StateMachine` class that handles the inputs and the state changes, and several state classes that implement the `State` interface.  
The interface is very simple and contains only one method

```java
interface State {
	  public State nextState();  
}
```
	
The loop of our Processing application is really simple too

```java
StateMachine sm = new StateMachine(initialstate);
void draw() {
  sm = sm.nextState();  
}
```

and this is the most basic implementation possible of the `StateMachine` class

```java
class StateMachine(State initialstate) {
  private State currentstate;
  
  StateMachine(State initialstate) {
    this.currentstate = initialstate;
  }
  
  public StateMachine nextState() {
    this.currentstate = this.currentstate.nextState();
   	return this; 
  }
}
```
	
Each class must implement the `nextState` method and return an istance of the next state that will be executed.  
With this knowledge in mind, this is how you build an infinite loop inside an inifinite loop

```java
class InfiniteLoopState implements State {
	public State nextState() {
	    return this;
  	}
}
```

But we can do better!  
How about a ping pong?  

```java
class Ping implements State {
	public State nextState() {
		println("ping?");
	    return new PongState();
  	}
}

class Pong implements State {
	public State nextState() {
		println("pong!");
	    return new PingState();
  	}
}
```
	
We moved the logic of the application out of the central `switch/case` statement in the `draw` function and deconstruted it into smaller pieces, that only know about themselves and the next state they are going to emit.  

As long as your state classes implement the `State` interface you can exapnd the concept to fit your needs.  
For example, if you need to keep track of the environment and/or the previous state, you can adjust the `State` interface to support it.  

```java
interface State {
	  public State nextState(State previousstate, StateMachine sm);  
	  // StateMachine holds the environment for us
}
```

and modify `StateMachine` accordingly

```java
public StateMachine nextState() {
  this.currentstate = this.currentstate.nextState(this.currentstate, this);
 	return this; 
}
```

**TL;DR:** state machines are easy, use them!

You can find examples on how to use this pattern and how to add more features in the [github repository](https://github.com/wstucco/processing_state_machine).
