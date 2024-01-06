---
layout: post
title:  "Does Functional Programming scare you?"
date:   2017-03-14 08:43:56 -0800
categories: medium blogs
--- 

Functional programming has been on rise of late. Boasting on so many amazing features in functional programming, a plethora of new set of functional programming languages have come to existence. All these newer languages have been trying to reduce the friction a programmer faces while trying to move to functional programming. With its unique set of features, many of them target a specific audience, e.g. Scala has been targeting Java and object oriented community by providing a unique combination of functional and object oriented philosophy on JVM. Elixir has been targeting Ruby community with its unique features of ruby like syntax and actor model based inter process communication built on top of Erlang VM and so on. There are many more languages and they provide some or the other features which seem difficult to resist.
> Even with so many attractive features, functional programming is still not so mainstream.  
  
The need and benefit of functional paradigm is beyond doubt and in fact it has become so obvious that even the traditional non functional languages like C++, Java and C# have added a lot of features to give a taste of functional paradigm to its user base.  
  
Although it seems super cool to read about these languages, getting started with functional languages have not been so easy, specially who have been programming in some traditional procedural or object oriented languages like C, C++, Java, Ruby etc for long.  
  
Functional paradigm is full of interesting but intimidating jargons like statelessness, higher order function, currying, monad, category theory and what not.
Although it’s quite powerful, yet it’s a big paradigm shift and needs a lot of learning and unlearning as well.  
  
Instead of focusing on complexity and jumping into unknowns, we can try to build on what we already know.  
  
In this series of blog posts, we will try to revisit some of the basic concepts from traditional programing languages( or general programming) and see how these evolve very differently in functional programming paradigm.  
  
Here we go:
  
**Recursion**  
  
There are problems where recursion is a very natural choice E.g. Fibonacci series problem or a problem involving tree traversal. The very definition of Fibonacci number, suggests a recursive solution  
 
{% highlight shell %}
f(n) = f(n-1) + f(n-2) 
{% endhighlight %} 
     
Same goes with Tree traversal.  
  
Almost every time you write a code using recursion in any moderately big project(in non functional language) you will be asked to revisit your code and write an equivalent iterative code. I, myself, have been asked couple of times. although many times it can be done very easily but sometimes an iterative solution may not be as simple as the recursive solution.  
  
**Why is recursion dreaded in imperative programming?**  
  
Whenever you call a function(and it does include calling itself) the return address and the arguments are pushed onto the call stack. The stack is finite, so if the recursion is too deep you’ll eventually run out of stack space. when your input is a variable and you don’t have control on it, typically you don’t want to take a chance.  
  
**Do functional languages handle it differently?**  
  
Won’t it be nice if the compiler somehow figures out to optimize the space required and we can write recursive code more often if it’s a natural fit?  
  
Most of the functional languages do exactly that. They use different techniques to optimize recursive calls to handle stack overflow issue. One of the techniques is Tail Call Optimization.  
  
Specifically if it’s a tail recursion(if the recursive call is the last call and there is no operation on the return value) they almost do this without allocating any extra stack space.  
  
Tail recursive version of Fibonacci number in C++

{% highlight ruby %}
int fib(int term, int val = 1, int prev = 0)
{
if(term == 0) return prev;
if(term == 1) return val;
return fib(term — 1, val+prev, val);
}
{% endhighlight %}    
  
Such code in functional languages are optimized and doesn’t need extra stack space.  
This enables recursion to be a natural fit for functional paradigm.  
So problems where recursion seems to be an obvious choice we don’t have to think and work extra hard to write iterative code.  

Have you tried writing a quick sort in iterative fashion? Now try the same with recursion and see how the approach changes.  

> **Recursion is at the heart of functional programming, so better be well versed with it.**  

**Array vs List**  

What’s the basic data structure C++ or Java provide? 

 > If you think carefully Array is the only data structure(and classes/struct) that has a native support in these languages. Everything else you have to write(or use the standard libraries that comes along with it).   

Array comes with a trivial but very important feature of random access.  
  
How do you traverse an array?  
  
Mostly you will use a loop construct. How do you sum all the elements of an array? Again using a loop construct?  
  
Now, lets change the problem a bit.  
  
Given a binary search tree, how would you traverse and print all the nodes? Will you use loop constructs or recursion?  
  
What about summing all the numbers in a BST?  
  
If you see the choice of constructs, it changes based on the data structure you are traversing. Although you can probably use recursion for array or non-recursive solution for BST, the solution won’t come very naturally.  
  
What about a linked list?  
  
How will you add all the numbers in a linked list?  
  
Can you think of list as recursive structure like an element pointing to another list?  
  
If you see a linked list as recursive structure you can apply some of the recursive approach for solving many of the problems.  
  
Sum of all numbers or multiplication or similar problems where we accumulate the data after iterating through each element and apply some function(like multiplication or square etc) is a very common task. All the functional languages provide kind of abstraction for these kind of operations called Reduce or Fold.  
  
Sum of all numbers of an array in Javascript:  
{% highlight javascript %}
var sum = [1, 2, 3].reduce(add, 0);
function add(a, b) {
return a + b;
}
console.log(sum); // 6
{% endhighlight %} 

**What about other common operations in array/list?**  
  
You might have observed, there are few operations that we do very frequently on an array/list e.g. selecting a subset of elements matching specific criteria, creating a new array out of original array by doing some standard operation on it e.g. Removing a new line characters from all the links read from a file, accumulating the result after applying some standard operation on each element e.g. Sum of the square of all the numbers or so on.  
  
These are so common that most of the languages provide a shortcut for doing this, so that we don’t have write our own recursive code all the time(many of these are of course implemented using recursion and list).
  
> Map, Reduce, Filter, Fold etc are some common constructs provided in functional languages that greatly reduce the necessity of a loop construct.  
  
**Pipeline, higher order functions, currying and more**  
  
So far we have covered only few of the trivial but fundamental concepts that will help us understand the Functional paradigm. I’ll cover many of the remaining concepts in the upcoming posts. Please stay tuned for part II.