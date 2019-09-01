--- 
title: "Top-Level Infrastructure"
date: "2019-09-01T21:19:39+08:00"
tags:
- adventures-in-motion-control
draft: true
---

As mentioned in [The Next Step][next-step], the first task will be to set up the
application's structure and define how the various components will communicate.

## Multitasking

Most embedded systems will implement multi-tasking by rapidly polling each
system within an infinite loop.

```rust
loop {
    poll_comms();
    poll_motion_planning();
    poll_io();
    poll_machine_events();
}
```

A motion controller will normally spend most of its time polling, but there are
places where polling isn't appropriate. For example, accurate movement of a
stepper motor relies on sending pulses at very precise times. Another scenario
is in the handling of communication, where waiting for the next poll to read a
byte may result in missing part of a message.

To deal with this embedded systems use [interrupts][interrupt], callbacks which
will preempt the normal flow of execution to handle an event. These callbacks,
referred to as *Interrupt Service Routines*, will then do the bare minimum
required to handle the event so execution can be resumed as quickly as
possible. 

For example, this may happen by saving the event to memory (e.g.
*"received `0x0A` over serial"*) so it can be handled by the appropriate
system when it is polled next.

## Inter-System Communication

Now we know that systems will do work by being polled frequently, lets think
about how they'll interact with the rest of the application. 

In general a system works by reading state, evaluating some logic using that
state, then propagate the results of that logic by sending messages to the
rest of the system (or even to the real world via IOs).

Systems will be polled infinitely and are the top-most level of logic so they
must handle every possible error, even if that is just done by entering a fault
state. It doesn't make sense for `poll()` to return an error (who would
handle the error?)... Or anything, for that matter.

From that, we've got a rough image of what a system may look like:

```rust
trait System<In, Out> {
    fn poll(&mut self, inputs: &In, outputs: &mut Out);
}
```

## Time

An important responsibility for a motion controller is being able to change
the value of something over time (hence the *motion* in *motion controller*).
Timing is used all over the place and each motion controller will have its
own way of tracking the time (remember that there may not necessarily be an
OS), so lets pull this out into its own trait.

```rust
use core::time::Duration;

trait Clock: Sync {
    /// The amount of time that has elapsed since some arbitrary point in time
    /// (e.g. when the program started).
    fn elapsed(&self) -> Duration;
}
```

{{% notice note %}}
This is some text!
{{% /notice %}}

> **Note:** A `Clock` will be shared by almost every system and a reference
> may be passed across threads. Hence the `&self` in `elapsed()` and the `Sync`
> bound.

Extracting the concept of time into its own trait is especially handy during 
testing or if you want time to run faster than 1 second per second.

```rust
use std::time::Instant;

#[derive(Debug)]
pub struct OsClock {
    started: Instant,
}

impl Clock for OsClock {
    fn elapsed(&self) -> Duration {
        self.started.elapsed()
    }
}
```

[next-step]: {{< ref "announcing-adventures-in-motion-control.md#the-next-step" >}}
[interrupt]: https://en.wikipedia.org/wiki/Interrupt