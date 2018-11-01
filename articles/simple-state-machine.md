---
layout: page
date:   2016-07-29 11:35:21 +0100
categories: c++ c++11 tuple fsm
---

# Simple state machine in C++ #

Some time ago we faced a small problem – we need a state machine in our embedded project. There are several
libraries and tools available out there, starting with things like [SMC](http://smc.sourceforge.net/) ending
on boost [MSM](http://www.boost.org/doc/libs/1_64_0/libs/msm/doc/HTML/index.html) library. Our application is
running on STM32F103 family microprocessor with 20kB of SRAM so size matters. :-) Using boost would be overkill because of few reasons:


* we want small and maintainable code also we want small output binary size. MSM is quite small but probably
[not small enough](http://www.boost.org/doc/libs/1_64_0/libs/msm/doc/HTML/ch04s04.html).
* we want zero dynamic allocations, reason for that is that our project needs to run on microprocessor for days
or months in very memory limited environment. I don’t want to care about tracing every memory allocation (it’s
hard to say anything about MSM at first glance but after quick “grep” we can find some std::deque’s and vectors in the implementation).
* the main requirement of such library is ability to “keep” current state and allow to trigger state transitions with some event
* not generated code is a plus, unfortunately SMC is “generator”

Keeping this in mind I’ve created very small state machine library consisting of only one header file and have
only dependency to great [meta programming library by Eric Niebler](https://github.com/ericniebler/meta).
The outcome of this idea is my [fsmpp library](https://github.com/lukaszgemborowski/fsmpp).
State machine description is inspired by boost msm but slightly different, for example:

{% highlight c++ %}
using my_transition_table = fsm::transitions<
    fsm::transition<StateA, TriggerA, StateB>,
    fsm::transition<StateB, TriggerB, StateC>
>;
{% endhighlight %}

Where StateA, StateB, StateC, TriggerA, TriggerB are user defined classed. Given such transition table you can create state machine such as:

{% highlight c++ %}
fsm::fsm<my_transition_table> sm;
{% endhighlight %}

This code will instantiate all state classes for you in std::tuple – you do not need to do anything special.
Using std::tuple is not requiring any dynamic allocation so state machine size is known at compile time. Each
state class has to have some special methods:

* enter() – called when entering the state
* exit() – called when exiting the state
* event() – called when handling event in current state

you need to define them, keep in mind that if your state is handling TriggerA or TriggerB you need to implement
appropriate event(const EventType &) method. Not doing this results in compile time error. To trigger state
transition just call fsm::on method on fsm instance, eg.:

{% highlight c++ %}
sm.on(TriggerA{});
{% endhighlight %}

being in StateA fsm will call StateA::event(TriggerA &) method. If the call return true state transition (in
this case to StateB) occurs. Meaning it will call StateA::exit() and then StateB::enter() methods. Full
example can be found in projects [tests directory](https://github.com/lukaszgemborowski/fsmpp/blob/master/tests/).

For now this is just proof of concept but is somehow usable.
