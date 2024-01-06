---
layout: post
title:  "A brief history of languages"
date:   2016-10-22 08:43:56 -0800
categories: medium blogs
--- 
  
**Why a new language every 6 months**  
  
“Hey, you know, Elixir is an amazing functional language. I am really liking it”, said an enthusiastic me to show off my knowledge about what’s happening around the tech world.  
  
Him: “ Ok. I am sure it’s. But, were you not a Ruby fan? You wanted to rewrite our complex code base in DSL created using Ruby. You showed a working prototype as well. It seemed quite impressive! Why a new language? I don’t understand why a new language is still coming year after years. Aren’t we having enough? C++, Java, Python, Ruby, Scala and what not.”  
  
Me: “Yeah, that’s true. But all these languages target a different audience, problem space and sometime built with a different philosophy. Developers’ world is fragmented. How people perceive the concept of complexity and challenges in software development is different. So people pick their choice of languages to solve their part of the problem using their choice of languages.”  
  
Him: “ I understand part of this. I can understand if you work on embedded system, you need a different beast vs when you work on higher level application, you may need different sort of power. But, what I see is so many languages in the same space.”  
  
Me: “ Yes, you are right. Sometimes there are too many languages targeting the same space. Some of these are because of the need of the hours and many times because of the battle of dominance between corporate houses. Each big corporation is trying to push for their version of languages to make sure they are not cornered. Sometimes the bureaucracy of the owners makes it harder to evolve the language at a pace and people tend to create their own solutions. At times people simply love the idea and go for it with a backing of a community.  
  
Him: “ I didn’t get it. Can you please be a bit more specific.”  
  
Me: “ Lets take a step back. We have been programming in C++ for years. We have a huge legacy code base in C++. Why do you think, we are moving towards Java.”  
  
**Broken promises and the blame game**  
  
Him: “ Java seems to be more productive. You write code in one environment(read Operating System) and you can run in another environment, without worrying about portability issues. IDE support is excellent. There is no pointer mess and no need to worry about memory any more. There are libraries for almost everything, you can think of. If you want logging you can use log4j, if you want injection you can use spring. You can find people from an amazing pool of talent, because almost everyone knows Java. Finding good talent in Java is so easy.”  
  
Me: “ Yes, it truly captures the essence of Java. We rewrote an app in Java, right? But the performance was not so good, I heard. It seems C++ equivalent was having 30k lines of code, Java is having 130k lines of code. It has only 15% of the capabilities yet and SLA has increased by a factor of 1.2. Now the new developers have to go through a bunch of starter kits to even start looking at the code, because the code starts from an xml and you don’t have a clue how the code reaches to that xml since the library we used for spring hides the details in some esoteric jars that you are supposed to be abstracted from.”  
  
Him: “ But that’s a bad example. You can’t blame Java for that. It’s just a bad engineering practices that created that mess. Someone thought everything should be configurable and they wrote huge xml files with configurations for even if you want to add a condition. The tasks, groups, beans, data pipelines, scratch pads and what not. It was a bad design. They did deep copy of huge datasets causing garbage collections to kickin so fast that 20% time is spent just in GC”  
  
Me: “ Wait! didn’t you say, in Java we don’t have to mess with pointers and memory anymore? Didn't you say, Java gave an amazing pool of talented engineers you can hire from and you have excellent IDE support?”  
  
Him, looking desperate to defend his position : “ yeah that’s true but if I think for us scala would have been a better choice. It’s functional, expressive, highly testable and composable and best part is it runs on JVM, so we can use the existing Java libraries. It has amazing community and people have built amazing high performing applications in a short time using Scala. It has an amazing actor model based framework called Akka and with that you can really build mostly stateless, purely functional, isolated components talking to each other just using messages. No more NPE in one component causing failures in other. They just talk in messages. If you crash, it’s just that you crash. Since the messaging contracts are very clear, you don’t need to worry about other components.”  
  
Me: “ Hmm. So you think Scala could be a good choice because of its functional paradigm?”  
  
Him: “Yes, I think so.”  
  
Me: “Interesting. It seems we digressed a lot from our original discussion of so many languages. Let’s talk about that now.”  
  
Him: “ Yes, very true. So my question was why do we need so many languages. I still read about a new language coming almost every 6 months. “  
  
**Complexity varies and so does the perception**  
  
Me: “ Yes, you are right. As you just explained why we moved from c++ to Java and why we should move to Scala, it actually tells a lot about the complexity of the applications and also what part we perceive as a challenge.”  
  
Him being clueless and trying to understand: “ I didn’t get it.”  
  
Me: “ See, you already talked about programming paradigms like object oriented and functional. So this is one of the factors. Lets talk about some other factors. What do you think our most of the c++ developers wasting their time on?”  
  
Him: “ Without a 2nd thought, it’s the mysterious compilation and linking errors. I actually sometime pity them. It’s year 2016 and they are still trying to figure out the meaning of unexpected token found in line 200, when it’s as simple as missing comma.”  
  
Me: “ Good. Lets talk about one more problem before we go back to our original discussion.”  
  
Him: “Sure”  
Me: “ Given a huge log file, if you have to quickly find the frequency of different error types, what will be your choice of tools.”  
  
Him: “ For such simple tasks, AWK will be just fine.”  
  
Me: “ Ok, and what will you do, if given a list of hundreds of users, you have to find their managers’ name from ldap?”  
  
Him: “This too is quite simple. I have done this before. I’ll simply use Python. Python already has amazing libraries to deal with ldap.”  
  
“I am also sure, you must be thinking of Ruby for this problem”, he said with big smile.  
  
“ Yes, of course . Ruby has been my go to scripting language of late”, I replied.  
  
**Search for a better abstraction**  
  
Me: “ So, I hope you understood, why I brought this discussion.”  
“ Languages are built on different philosophies and considering different kind of people in mind. You can think like this:  
  
People thought assembly is too low level so they wrote C. Which is still close to machine but provides more human understandable constructs. ‘for’ and ‘while’ are more readable than ‘go to’ and ‘move’ any day. Then came a time when projects started becoming bigger and managing code base became difficult. Thousands of methods spread across multiple files, with no overloading method names started reflecting individual personalities, populate vs fill, copy vs assign, add , addition, sum, summation etc started becoming common. Finding a method which does the desired thing became very difficult. We were missing something to group similar things together. Namespace was one way, but that was still not sufficient. We were looking for stronger cohesive forces which look more natural to bind these methods together. Concepts evolved and we came up with Object Oriented programming where data and its operations were bound together. Inspired from nature, Object Oriented concepts seemed to be a great way to abstract the complexities as it’s closer to daily life of an average person. Seeing a car and of make Honda, we can very well relate what a class and a object could mean. With that came languages like C++, Java.”  
  
Him, beaming with confidence, as if this is what he wanted to listen to: “ Yes, that’s true. OO paradigm still seems to be the most prevalent one. That’s why I love Java. Everything is a class unlike c++ which has some global method concepts.”  
  
Me: “Yes, it definitely started improving the scenario. People were able to write much larger code base now. But soon the euphoria ended when people started struggling with class design. Gradually, problem space itself moved to more higher level applications. Instead of writing automation for machine, now we were discussing how to send money over internet. how to define class for payment vs refund vs auth vs chargeback, things started looking very different from the toy examples of Animal and Monkey or Car and Honda and CRV.”  
  
Him, slightly disappointed : “Yes, I agree, the noun and verb classification doesn’t work very well. People became innovative and started writing noun form of verb to represent class. We have classes like Analyzer, validator etc and figuring out blueprint vs the implementation is fuzzy.”  
  
**Developer productivity takes the center stage**  
  
Me: “ With its shortcoming as well, OO helped big time. Over a period of time, problem space changed again. Machines also became much powerful, memory cheaper. With that we could leave with some non-optimized memory usages, few extra bytes won’t hurt at the cost of more productivity. It brought 2 interesting changes: dealing with memory appeared to be the most counter productive during application development and so this whole set of languages were born, which abstracts out the memory management part from the programmer, Managed languages like Java, C#. Go, rust and others just followed the path. Garbage collection became a hot topic and it still is.  
Another set of change occurred because of internet. We wanted code which can be written in one place but could run anywhere and so came languages compiling/interpreted into an intermediate languages and running on VMs. C#, Java followed the suite.”  
  
With a chance to prove the superiority once again, him : “See, I told you, that’s why I love Java, everything you mentioned indicates we should use Java.”  
  
Me: “While all these were going on, there were people who believed most of the time we spent on is compiling and linking. Why waste time in figuring out whether to use int or long or float, why doesn’t my language figure it out automatically. Why give an error for a method which I am yet to call. Why to write my own implementation of stack, hash, array for storing data when I know for sure I can’t write any useful program without it. With these things in mind, we conceptualized dynamic typing, duck typing, higher level of data structure and other constructs. Enter the age of Interpreter based languages promising everything we wanted for productivity.  
  
Perl, Python, Ruby followed the suite.  
  
While Pythons’ philosophy was based on readability, one and only one standard way of doing things, Ruby believed in giving power to its user. The magical power with which you can generate code, hide abstractions and actually create a new language all together(read dsl).”  
  
Him: “ I know. These languages are productive for a short span. But I have heard these are much slower and the worst of it, we throw the errors directly on the face of customers.”  
  
**Reusable later, usable first**  
  
Me: “Yes, you are right. It is typically slower, and you do miss an important error check step during compilation. But have you wondered, a startup trying to build some working prototype, where would they prefer to spend time, showing off what all capabilities they can provide or struggling to decide whether to use long or double, which libraries to use to parse JSON. It doesn’t matter how scalable or fast you application can run in future, when it doesn’t work now when you have to give a demo to your investors.”  
  
**Dealing with complexity**  
  
“While all these adventures were going on, there were another effort on functional programming. The idea was simple. Programs are nothing but manipulation of data and what could be a better way of manipulating data than mathematical functions. Pure methods, immutable data, side effect free programming became a theme for these guys. Lisp, Haskell, Erlang followed the suite. These languages believed the most counter productive things programmer do is to deal with the untraceable, unintended state changes and then debugging through the code to find where the variable was updated. Erlang went a step further and made sure only way two processes(erlang processes) can talk is through message passing to make sure one part of the system doesn’t corrupt the other. People built applications with nine 9s availability with erlang( yes, it’s nine 9s, 99.9999999)  
  
“Some of these languages provided a way to modify the underlying AST and the language constructs to build a completely different beast from the same language. Lisp’s macro, Ruby’s metaprogramming falls in these categories.”  
  
**End of Moore’s law**  
  
“Early 21st century brought in some major changes in the industry. Moore’s law was no more applicable. Processing power of a chip was not increasing at the same rate at was in past two decades. Instead of more powerful cores we started scaling horizontally by adding more cores. Internet brought a scale to applications which was earlier only a problem for telecom domain. With a goal to web-scale things had to change.”  
  
“ Parallelism, distributed, concurrency seemed to become the de facto buzz words. So new languages had to evolve to keep these things in mind. Go, Rust, Nim followed the suite.”  
  
**It's not a zero sum game, let’s collaborate to win**  
  
“Over the period of time, people realized one or other philosophy may not win and we may need to support multiple paradigms. People also realized we can’t get away with the legacy code. The amount of investment is gone into these legacy code we can not simply let it go and start from scratch. So new languages started targeting portability and multiple paradigms.  

Scala e.g targeted JVM with an option of functional and object oriented paradigm. Clojure targeted JVM with lisp like macros and functional approach. Elixir targeted Erlang VM with a Ruby like syntax, actor model and functional approach.  

Crystal aimed for c like performance with Ruby syntax. Nim aimed for a system programming with more powerful macro, concurrency and parallelism support and interoperability with c and other programming languages.”  
  
“Another interesting language is Lua, which aimed to be an scripting language embedded in another language. This gave an amazing performant customization options for existing applications. You can add customization to existing c/C++ applications like HAProxy, NGINX , redis and many more with Lua. It helps you focus on your customization rather than building the whole stack by yourself.” I continued.  
  
Him, amused but looking desperate to catch a breathe: “ interesting. If so many options and choices are there why do we still suck at development?”  
  
**Choices alone don't help**  
  
Me: “Irony of choices is mostly we pick up the wrong choice. Instead of looking at the problem at hand we look at the solutions first and become too attached to it. Jingoism around the languages do not make it better either. Picking up C++ for web development is as little helpful as picking Java for network programming. Both of them actually can do either of it but why to pick hammer when problem is not nail?”  
  
Him, with a sudden realization: “Hey, you didn’t mention JavaScript at all?”  
  
**Front end a different beast**  
  
Me: “Yes, we didn’t talk about JavaScript and a whole lot of languages in that space.”  
  
Him, this time even more surprised: “ what!! We have many languages in that small space too?”  
  
Me: “ Yes, by choice or eventuality we are stuck with JavaScript on front end because of backward compatibility. People got so sick of JavaScript that they created languages which will generate JavaScript . You might have heard of s Coffescript, typescript etc. the idea here was to reduce the development/debugging/error handling limitations created by JavaScript. Some of it gave functional touch to it and some gave a static/dynamic typing to it. Elm went one step further and not only targeted JavaScript but the whole echo system of html, CSS and JavaScript.”  
  
“We didn’t talk about some other platform specific popular choices as well. If you on Windows stack if you some fabulous languages at your disposal. C# is amazing, you feel at home when coming from c++, all the problems are handpicked and resolved e.g. Property( works like functions, used as variables), references input output, delegates, linq and the best part is tooling around it. Visual studio is the best IDE we have ever seen. Same goes with F# its functional equivalent.    
Same goes with Apple echosystem objective c and swift are amazing at what it does.  
  
Him: “ with so many choices how do we pick the right one?”  
  
**Making a good choice needs some effort**  
  
Me: “ it’s is complicated but not as complicated as you think. Look at your requirements. What’s your problem space? Embedded or web or backend. What’s the scale you are targeting? How many people are going to work on this? Do you have memory constrained(remember we have resources may be cheaper but neither free nor infinite). Is it targeted for windows, Linux or iOS/Mac? What kind of users are going to use it: average end user or programmer or product managers? What kind of throughput you are expecting? Is it going to run on few boxes(e.g. Tools) or many? Do you have incentive to make use of multiple cores? Do you already have exposure to some languages that may be a potential fit? Do you have a legacy code base that you may want to reuse? Does your echosystem(logging, monitoring, data mining, service discovery, caching etc) support different languages or how long will it take to build them”  
  
Him: “ Hmm. So many aspects to look for. I didn’t realize choosing a language is so damn complicated. What’s your favorite among them?”  
  
**Needs define the choices, not the jingoism**  
  
Me: “Interesting! c++ is what I have done most of my career, so if it fits the requirements I’ll go for it. I have done some Java as well so that would be next if we have to work with a larger team, higher level applications or not so low lever stuff. For tools and internal projects Ruby is my go to option as writing code here makes you feel happier. The DSLs you can buil looks so amazing you’ll fall in love with it everyday. For small customization to an existing complicated application Lua is the one I’ll prefer. Writing an L4/l7 custom router or custom proxy server will need so much of effort, I’ll prefer using an existing proven technology like NGINX and customize using lua so that we get the same performance but customized behaviors. When I have to extract all the juice of a machine and build fault tolerant system I will go for Elixir. State less manipulation of data, lightweight processes, smooth concurrency support, actor model based message passing, in built constructs for fault tolerant, distributed system, hot code swap, in memory key value store and battle tested Erlang vm makes your life so at home.
For front end I tried JavaScript for a while but am tired of its bad parts(obviously it has its good parts as well), so wanted to give Elm a try which seems promising. Functional at core, targeting HTML, JavaScript and CSS together, amazing compile time messaging and time debugger, it makes front end programming interesting, fun and as close as back end programming " 
  
Him: “so much of talk, why no picture?”  
  
Me: “Because I am lazy to do it!!"  
  