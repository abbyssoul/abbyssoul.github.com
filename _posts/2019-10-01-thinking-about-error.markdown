---
layout: post
title:  "How to think about errors in your code"
date:   2019-10-01 12:00:00 +1000
categories: errors
---

Let us talk a bit about what is an error in a software engineering context. And more importantly, what can we do when one happens.
Have you ever discussed with your colleague if this particular case should be handled as an exception or if-else condition?
Or have you heard someone saying - "this case should never happen in production because of reasons..."?
As a software engineer, I spend a lot of time considering how particular inputs and situations should be handled. Is it an error? It is an expected situation?
Here I am going to share an approach I settled on when designing a libraries: [libstyxe][libstyxe-git], [libsolace][libsolace-git], etc.

Generally, when talking with colleagues with some engineering experience - there is always a consensus that errors _do happen_. There is, however, rarely an agreement of what an error is?

# What is an error?
It might seems like an odd question to ask: "what is an error?". Surely we all _know_ one when we see it. And why would you have errors in your code in the first place? Shouldn't our programs be error-free?
Yeah - real-world questions here.
One way to think about a program is like _a sequence of action_ to achieve some _goal_. For example: compute the result of a division of _two variables_;
_play_ the content of _that_ audio file; _display_ pictures of my favorite cat I have saved _over there_; let me _enter_ this data in _a file for the record_ and _create_ _a report_ based on that data.

Notice something common - unlike _infamous_ "hello world" program that takes _no input_ and always produces the same _result_ no matter what - programs in examples are designed to solve real problems (such as displaying cat pictures!) -
Although to be useful they all have some flexibility. That is - it is possible to use the same program to display dog pictures also. To do that - a program needs to take some _input_. After all, we may like different cats. Or music. Or have different business data. So different users have different inputs and expect different output - most of the time.
This extra freedom - to operate on a user-provided data - comes with a price. It is possible that user inputs may be 'invalid' in some sense? Division by zero, a filename that does not exist, none existent cat species (Oh-NO!)
What a program is to do in that case? Is that a program's fault/error?

Now it is all fun and games - when you deal with simple programs designed to [do one thing][unix-philosophy] and operating with a single user. How about something more challenging: a service that takes input from
100s, millions, billions of request. How about banking service? Airline departure control systems? Any issue with a user input gets that much worse.

To make matters even worse - it is not always a single request with invalid input that is immediately noticed. Remember _a_sequence of action_: take money from account A (done), add money to account B. Boom! Account B does not exist. Error!
And not money on account A anymore.

Surprisingly it actually can get even hairier when we consider sequences across multiple systems - distributed across multiple machines in different data centers.


# Brief overview of errors

When researching a topic of error handling for a proper article, the best I came across was a classification of errors based on when they happen. Syntax, runtime, and logic errors.
This is repeated across a countless number of blogs. I found this misleading and most unhelpful. For one - if this is the orderer list - logic error should precede compile-time errors :) You need to make a
logic error first then implement it for it be compiled. This whole false classification reminded me about [taxonomy of animals][classification-animals]. But jokes aside - this is not helping anyone because it gives no direction on how to approach errors.
This is why I wanted to first review this common classification before moving to what is actually important for any production system.

_Logic errors_ are not computer related per se. It may be a failure to _analyze_ the problem domain. False/noisy data. Or straight _blunder_. And let us not forget that it is possible to commit errors while writing code.
There is little _a software_ can do if it is implemented with a flaw. Good processes and procedures tend to help to address logic error. RFCs, documentation and peer review - are all good tools to make sure requirements are valid and logical.
The more people with domain knowledge check the specification - the higher the chance of it being valid.
And more people review the implementation of that specification - more likely it to be valid.
(Although on this note I should mention a personal anecdote - when particular changes were spec'd and reviewed, PR reviewed and approved - only to accidentally remove a good chunk of critical, but unrelated functionality. 5 people reviewed and approved. 2 spec'd and we still ended up with an incident in production. We had to revert this changes urgently. So it is not always about quantity, but engaging people with DOMAIN knowledge early.)

_Syntax error_ - is an attempt to pass to a compiler an ill-formed program. This is the reason why compiled languages exist. That and performance. - To prevent ill-formed programs from being executed.
Modern compilers do an outstanding job at that. So much so - that it is a good technique to leverage type systems such that a logic/coding error results in an ill-formed program caught by a compiler.
In case a non-compiled language is used - it is still possible to make a syntax error and have your program running. Until execution path hits ill-formed block that is. The advice here - have extensive _unit tests_. If there is a piece of code that is not exercised by a unit test - chances are there is a syntax error hiding there. Assume program does not work until proven otherwise.

_Category of runtime errors_ - is the broadest one. And most [expensive][cost-of-errors]. If we understand the problem and able to implement a solution as a well-formed program - we are free of the first two 'classes' of error. But it is runtime errors in your system that prevent users from using your software, what customers complain about on forums. It is the type of error that wake you up at night - when your service is down. This is the type that of interest to me here.


There is also an important dimension of this classification that is not always mentioned. Frequency and severity of the error happening.

# Thinking about errors

All the above classification is of little help when it comes to handing the actual error in runtime. So what can we do?
In order to identify what actions can be taken in case of an error occurred, let's consider different types of applications. There are 'hello world', and real-world applications that some take input and produce some output.
The input may be indirectly (command line, config file, script) provided or directly - read from keyboard, mouse, gamepad.
It can come from a network. Or - to generalize here a bit - a sensor: camera, keyboard, network device, IOT type sensor.
Real word applications tend to produce some output that depends on the input. Response to network messages, display picture, write converted file - etc.
This gives us an interesting concept to work with: a valid input _should_ produce valid output. If it is not the case - a program itself is incorrect. There is a bug in the application that must be fixed.
The focus of this article is to explore the failure of the valid program because using QA techniques such as testing - it should be possible to determine if a program is valid. That is given a valid input sample it should be possible to establish if
it results in a valid output. That does not guaranty, of course, that we have no bugs and all possible inputs won't uncover bugs. That is why it's important to have a representative sample of the input to validate the application.

Failure to _obtain_ an input (valid or not) thus must results in a failure to produce valid output. Of course, it is also possible to fail to 'store' the output even given valid input. That is - display has been unplugged just when that can picture was about to render. Or more realistically - network drive crashed when the video converter was writing the output. The network dropped in the middle of an upload.
It is also possible to fail to produce an output given valid input and valid output device - if the process failed to secure intermediary resources required to process the input. Out of memory to store the input etc. What makes this one particularly annoying is the fact that - resources required for processing can not always be predicted as it depends on the input itself. For example, if you want to view a raw uncompressed picture 8k picture of a cat - you might need a lot of RAM to fit the file in. Or a special app that can handle mem-mapped files.


So plenty of ways to fail. On the upside - understanding where failure can happen is critical to decide what to do about it.


## What do we do when a program fails?
Now that we know where valid bug-free programs fail - what options do we have?
Turns out "it depends on the app". It is one thing when the picture viewer app can't find a file name because of a type. The app can quit and restart. It is a different case when a server receives a bad request. It is not a good idea to close the whole service for other users. What about critical systems where failure is not an option?
Let us explore types of applications to see if we can generalize something.

The way I come to think about apps is - it is either a '_regular_' application or it is a '_service_'. For the purpose of error handling, we will have to (re)define the scope of what that actually means in terms of IO.
_Regular_ - means applications that have all the data they need to perform an action - whatever it is - readily available. This are 'simple' [Unix-philosophy][unix-philosophy] apps.
`ls /`, `rm -rf /`, etc.
`magick convert rose.jpg rose.zng`

We can observe that these applications do have their input given at the start. It could have been a config file or a script. The point here is that the input is specified. The only problem is - it can be invalid: `zng` for example is invalid file format, file `rose.jpg` does not exist in a current directory. The output is also defined. Store results in a file. It is possible you don't have write permissions. Or disk space. In any case - an error here - is the failure to acquire input or output resources. And the way to resolve it - is to correct inputs.  The only interesting twist is that if you specify the output device as part of your input - this is an input error. Confusing :)
An example of an output error - is if you print something and printer jammed in the middle of a print. (note to self: does anybody still prints things?). Ok maybe - `netcat <remote_ip>` and drop WiFi in the middle of transmission. The point is - output can fail too.
So what we do in this case - we try again. That is to say that the whole _process_ can be terminated at the point of error. Important note: all the inputs are immediately available. So a process can be restarted. In most cases it is also reasonable to assume that no output has been persisted as error prevented an app from producing one.  
(yeah yeah I know - `netcat` can read from stdin, or a pipe with ephemeral results - this was just to illustrate a point)

A different story when a process is responsible for the acquisition of data and its transformations. _Single player games_ come to mind as examples most users may be familiar with. A player smashing controls and some action takes place on the screen. Although in this case, we don't expect a user to press an unsupported button.
A service or job queue worker is a different example of the same concept - the input is acquired from some kind of a source is not known beforehand. In this case, input itself can be an invalid message. But it should not result in interruption of service.
Indeed it would be a pretty bad service if one user generated a message that brings a service down for everyone.
Thus in the model of message/transaction processing, it is possible and beneficial to isolate failures. One can think about isolation as each message/transaction being processed by a separate process. If the input is invalid - good old fail with error message strategy works _AND_ it does not affect other transactions. In fact, this model has been implemented in `inetd` and an early version of `Apache` web server - where each request has been handled by a new process. The downside is
spawning a new process for each request takes too long and each process has memory overhead. Some improvement of that approach is a pre-fork model - where a number of processes created beforehand in standby mode. When a request is received - it is handed off to a free process/worker. Number of worker-processes can be monitored and when one crasher due to bugs of input issues - a new one spawned in its place.
In order to fight memory overhead of processes - some servers chose to handle requests in threads. In fact, most modern web-servers are threading based.


# Conclusion

In this post, I tried to untangle the mess of error handling. Going beyond simple bugs in the implementation, errors can occur in a course of normal operation. Failures DO HAPPEN and an application must have a strategy to detail with it.
Crashing the app may be a perfectly valid way for simple apps. In case the app does not deal with input acquisition itself but gets all it needs from the start. User-facing apps can produce: "Invalid input - correct and restart".
The interactive app has an option to maintain a dialogue. You don't want to crush the whole app if you walked a user from a serious of actions only to start again. Maybe just this one input.
Services take this step further - they have multiple independent inputs and in case one was invalid - it should not prevent others from being processed.
This was more of an overview of what is there. I wonder if I missed some important aspect of error handling here?


[classification-animals]: https://en.wikipedia.org/wiki/Celestial_Emporium_of_Benevolent_Knowledge
[cost-of-errors]: https://medium.com/@ryancohane/financial-cost-of-software-bugs-51b4d193f107
[libstyxe-git]: https://github.com/abbyssoul/libstyxe
[libsolace-git]: https://github.com/abbyssoul/libsolace
[unix-philosophy]: https://en.wikipedia.org/wiki/Unix_philosophy
