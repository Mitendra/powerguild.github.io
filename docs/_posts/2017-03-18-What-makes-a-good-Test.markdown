---
layout: post
title:  "What makes a good test?"
date:   2017-03-18 08:43:56 -0800
categories: medium blogs
--- 
  
![](/assets/images/laptiop_open.png)  
      
Test cases are meant to check the health of the system and find issues if certain part of the system is not working as expected. Although this seems pretty obvious and trivial but in practice it’s still something engineering teams struggle to achieve with ease. There would be scenarios when the tests succeed but system is not healthy and similarly failing tests do not always reflect a problem in the system. Although all tests are mostly written with good intent, all of them don’t turn out to be good one. Many of them actually becomes useless over a period of time.  
  
So the question arises, what makes a test good or bad. Lets look at some of the signs of a good test:  
  
## A good test is deterministic in nature**  
  
Success or failure of a test case should be able to tell whether a system or part of the system is good or bad with confidence. But isn’t this very obvious? Actually it is, but writing a test which is deterministic is not always easy.  
  
**What could make a test non deterministic?**    
  
1. Test data randomness  
  
![](/assets/images/random-dice.png)  
  
Randomness of data is preferred in many cases. One of the most dominant reason for doing so is to reduce overall runtime. In theory you may have thousands of combinations of test data and ideally you would like to test everything. But tests are also resource intensive, it consumes CPU cycles, memory and most importantly time.  
  
Randomization is a way to pick certain variation of test data randomly and use them. E.g. Lets say you are planning to test a credit card payment flow. Your primary objective is to test some behavior of payment. Now you have a number of credit cards to choose from Visa/MasterCard/Discover/Amex and so on. You don’t expect credit card types to impact the behavior but you are not sure enough. One option is to write variation of same test case with all the credit cards. This way you will be able to tell the behavior for each use case. But since you don’t expect it to impact the behavior you don’t really want to waste 8–10 times more resource and time for testing something that may not find anything different. Instead you choose the other option, where on every run of the test case you pick a random card. That way over a period of time you test all the options and also not waste a lot of resources. If some cards happen to impact the behavior, next time you just add that specific card test case as a separate test. This is one of the very common practice in industry.  

> A good test framework provides a way to repeat the same test with exact same values, even in the first place it was random.  
  
Most of the framework uses something called seed to keep a track of what random value is being used. So the seed itself gets generated randomly but if a specific seed results into issue and you want to rerun the test for the same behavior you simply pass the same seed as in last test as input to your test(seeds typically are optional) and it produces exact same data as in the previous test. So although the test data generation is random but it provides a way to regenerate the exact same input set by passing the seed.  
  
That’s how it achieves deterministic nature although being random actually. 
  
2. Tests affecting the state of the system  

![](/assets/images/tension-among-states.png)     
  
Tests are written typically to work in isolation but the underlying system under test may change its state based on the test execution. So when you run the next test previous tests data may impact the result of next test.  
E.g. If you have test for doing a payment from account A to B, after the test account A’s balance may get updated, so if subsequent test case uses the same account it may not have enough balance to do the payment.  
  
This brings us to some interesting options:  
i. Order of test cases: many times we try to make sure test cases which can impact each other are run in an order that the system status doesn’t change the overall run status.  
ii. Every test case having its own setup and Tear down: this makes sure all test cases start on a clean slate.  
  
Option ii) seems quite reasonable but is often very difficult to maintain for several reasons:
a. It’s resource and time intensive: to do the whole setup like creating an account, adding funding instruments etc add much more overhead for the test.    
  
b. Knowing the whole system is difficult: while some well known stuffs could be very well isolated but the challenge is to know all such status change in a large application with hundreds of subsystem is not easy. E.g. Even though it seems creating separate account is good enough step for test isolation. We don’t really know if many similar account would or wouldn’t create a skewness in test data. Production systems are typically written for practice use cases but in test environments data may be a bit biased. E.g. In practice if you take the birthdays of all the account holders, there will be very well distributed in a very wide range. But in test environment you may be creating accounts with only few birth dates or in many case having the same birth date. This skews the data. So if you try to load the data using such skewed key the load performance will get worse over the time as you run many re and more tests, resulting into non deterministic behavior of test. Running the same test at two different times may result into different result.  
  
3. External system affecting the tests    
  
There are tests or system under test which may interact with external systems. Some of them may be unavailable at times resulting into test behavior change.  
  
4. Parallel test execution  
    
For faster test execution, quite often, the tests are run in parallel. While most of the time it’s not a problem but many times they can interfere with other. There could be multiple factors impacting the behavior like test data, scalability of the test framework itself and some time underlying system under test itself may not be able to handle the use cases in parallel.  
  
## A good test completes quickly and consumes less resources  
  
The number of tests grows exponentially with every new feature that gets added in the product. Over a period of time it grows so much that running all of them for every change becomes too time/resource consuming and often because of time pressure many of them are not even run. A less resource and time consuming test comes very handy and those are the ones which will last much longer.  
  
## A good test not only tells there is a problem but also narrows down to the specific component/feature or code path  
  
![](/assets/images/dont_show_tell_me.png)  
    
Even though the objective of a test is to identify a systemic issue but the larger goal of having a test is to help identify the broken piece and get it fixed quickly. If a test failure is very generic it becomes very difficult to debug and find the actual issues resulting into delayed delivery. The test failure points should be focused, specific and should contains enough information to debug the issue.  

## A good test is easy to maintain  
  
![](/assets/images/easy_to_maintain.png)  
  
The life expectancy of a test code is much higher than the actual product code. So a large part of that remains in maintenance mode. Changes in product behavior result into corresponding change in test behavior. Such change in behavior causes changes in test code quite frequently. The changes could be as simple as change of some expected response code or it could be as huge as change of whole request/ response or sometime making the test code completely redundant. Since the changes are typically quite frequent it’s important these can be done quickly. Any test where maintenance cost is high is doomed to be skipped or ignored pretty soon.  
  
## A good test is easy to run against different environment with minimal setup  
  
Engineer’s are generally pretty much emotional about the languages and tools they use. The larger the team is chance of having different preferences is higher. With so many choices of tools and environment, it’s important that all of them are able to achieve the same goal of delivering high quality product without stressing out. Some one may prefer to run the tests on their desktop while many would want to run them as scheduled Jenkin jobs. The easier it’s to run a test case with minimal setup the chances of it being run is higher. Similarly more complex setup drives an urge to ignore such test execution affecting the quality overall.  

## A good test is easy to find based on the use case and failure points  
  
![](/assets/images/find_user.png) 

While most of the time tests are run to find the issue, many times it’s the other way round. You find an issue, fix it and now want to verify if the fix is correct or not. If a test is not easy to find based on the use case, many times people tend to run the whole test suite, which is not only time and resource consuming it doesn’t give enough confidence whether the required verification is done or not. This fear causes people to run even larger set of tests causing a significant overhead in delivering the fix. An easily discoverable test is the one which are typically run more often and catches more issues.  
  
If you have been following along you may have realized now, why writing a good test is not easy. You may also have observed many times it’s not just the test in isolation, but the tools, framework and environment that is used, together make a recipe of a great test.