---
layout: post
title:  "From startup to web scale: A gentle introduction to scaling"
date:   2018-06-19 08:43:56 -0800
categories: medium blogs
---  
  
![](/assets/images/highrise.png)  
  

This post is a gentle introduction to the concepts of scaling a web service and different phases it goes through. Subsequent posts will have more details about many of these concepts introduced here. 
  
**Inception** 
   
One fine day, you think about an idea of creating a website which will help sell products online. You know a few programming languages that you have studied in your school, you also have a basic knowledge of database and you can quickly learn few hacks to put together a webpage. You spend few hours thinking about the use cases, designing the database tables, implementing the details, creating a front facing UI to interact with the app, do basic testing and host the app for public usage.
  
You share the details with your friends, many of them like it and start using it. You start seeing a trickling traffic.  
  
**Curiosity to know how it’s doing and the analytics**  
  
You know the database details, so you do few queries to find how many people used it what kind of stuff people are interested in. But, querying it every time is hard. So you start putting together an analytics page where you can get these detail easily.
Growth in user base and one off errors
The app becomes popular. More traffic is coming in. Many of your friends complains that they see few errors randomly.You realized you haven’t added much logging and there is no easy way to find what issues they are facing.  
  
**Logging and debuuging**  
  
You put together a basic logging strategy. You write most of the logs to file and in certain scenarios to database itself to collect information later.  
  
**Monitoring and alerting**  
  
You have logged enough and are waiting for next set of failures. The error doesn’t surface for few hours. You keep on looking at logs periodically, but it simply refuses to come back. So, now you write another script which will keep monitoring the log and send an email if any issue occurs. After few hours, you get the first alert. There is enough details to fix the corner case and in next few hours the needed code is rolled out. You are about to tell your friend that you have fixed the issue, but you got a pleasant surprise. The friend himself is calling you. It seems either he himself figured out or this is what probably is called telepathy. With the excitement you pick up the call, but it seems there is more bad news.The friend tells you that earlier only few things were randomly failing but now nothing works and the server seems to be down.
Availavility issues and planning for high availablity
It seems while you were trying to roll out the code, the server went down. It sounds so stupid, but you just did it. You want to quickly get out of this mess. You decide to have a basic redundancy in place before adding any new feature.  
  
**Load balancing and deployment strategy**  
  
You strategy is pretty simple, you want to host the server/app on multiple boxes. You have heard about nginx. You quickly setup an nginx reverse proxy and couple of boxes behind the proxy. You also decide that next time onwards for upgrading the code you’ll take one box out of traffic by ECVing out the box and then upgrade the box and put it back in traffic and repeat the same for the other boxes.
  
This solves your current issue. But you don’t want to just stop at the current issue and want to figure out what else needs redundency. You didn’t have to think much, you know database is the other major chocking point.  

**DB replication, periodic backup and fall-back setup**  
  
So you decide to take periodic backups of the database and also set a fallback redundant db setup.  
  
All these required few simple scripts and you just added these automation scripts to the same code base.  
  
**Aging code base and need for refactoring**  
  
With all these code getting added to the same place, code has started growing. You have got few separate stuffs going on in the code: deployment scripts, monitoring and alerting scripts, analytics queries, user interface, admin functionality and your main feature app. The codebase has started to show its age.
  
Before it’s too late, you want to split the code base based on the functionalities and with that you start putting them in separate folder and naming those according to the domain it belongs.
  
Your friends are happy. Everyone is giving a thumbs up.  
  
**More features and further growth**  
  
Your app continues to gain more and more traction and high number of hits. Customers are asking for small but many new features. Customer focus has always been your strength. You want to make sure each one of them are satisfied. You have been taking care of all the small and big features they have been asking for.
  
Slowly but surely, your code has been becoming messier.
  
**Broken features, testing strategy and environment**
  
With several feature releases, it’s become difficult to think through all the use cases. Silly mistakes have started popping up frequently, breaking major features. Unhappy customer are what you don’t like. It’s time to focus on best engineering practices. While you have already been doing unit testing and a bit of functional testing; end to end testing has been missing. You decide to write multiple end to end tests which will cover the whole user experience. You realized, these tests need db setup, server setup and more. It’s time to invest in test environment, where you can replicate the whole production setup and run tests. A good testing environment and solid testing strategy is showing the result. Customers are happy again.
  
Happy customers mean more opportunity and more asks.  
  
**Further growth**  
  
More and more new features are being asked. There are requirements for sending end-of-the-day report, monthly reports, emails, texts etc. Few features need integration with payments processor, accounting and more.  
  
**Domain isolation**  
  
With so many different features, maintaining the code base has become difficult. You have hired several folks to help you. Lot of communication gap and people are stepping on each other’s toes. There are different areas of expertise like transactions, UI, accounting and so on.
You decide to break the app into multiple smaller apps. Each focused on one area or a domain.
These apps would have separate code base and would be owned by different developers/teams. You decide that all these apps needed to be deployed and run independently.  
  
**Services and service contracts**  
  
With multiple apps, you want to make sure end to end functionality is working correctly. So you concluded to have a contract between these services and all these services are supposed to honor the contracts strictly.  
  
**Asynchronous tasks, batch and daemons**  
  
There are also asks for offline reports and periodic emails. You want to write batches which will run periodically. Few batches will be running once in day , few more frequently and few less frequently. You also have a requirement where the tasks that take longer to complete need to be done asynchronously. You don’t want your customers to stare at the spinning wheel on the screen. So, you start investing in daemons and messaging queues. Services will do the basic validations, make sure the requests could be honored and do the basic set of mandatory steps and then instead of processing non essential stuff immediately(like sending email, texts etc), it will serialize the request in a specific format and add them to the queue and respond back with a status success and pending for completion of non essential stuffs. The UI will use this response for indicating customers that the request has been processed successfully and texts, emails will be processed asynchronously later. Daemons will pick them and process the long running activity and non essential stuffs later. It will also push the result of the operations to email/mobile text and update in the db. UIs will pick the latest state from the DB, So customers won’t have to wait for the result.
  
Once all these basis architectural changes are done, things look pretty good.  
  
There are few complaints here and there about slow responses but overall things look solid.  
  
**Load issues and db scaling**  
  
With initial few complaints, you start exploring what can be improved to get rid of such complaints. You start with basic query optimization, but as the time goes, the problem of slow responses started happening more frequently.  
  
**Indexing, partitioning and archiving**  
  
Careful analysis showed, a fewbad queries are doing full table scan and you are quick to figure out right indexes would help. Gradually it is obvious that the usage growth is tremendous and just indexing won’t help. You realized that you still have all the data since the start of the app in the same db and cleaning up may be an option. So, you decided to partition the data. On few tables you went ahead with monthly partition and for few you went for a daily partition. You also decided to put a data retention and archival policy.
  
There were few hiccups during the implementation but it was worth taking the chance.
  
With reduced data volume per partition queries have become much faster and you again have happy customers. This also means the growth trend continues.  
  
**DB split**  
  
With continuous growth, the data volume has again started becoming the chocking point. Even after query tuning, optimization and hardware upgrades there are no other choices left but to split the database.
  
While earlier db optimizations were much isolated in nature and didn’t impact other part of the code, this time it is not the same.
  
Most of code assumes, all the tables are in the same database so you have never worried about the transactionality, database is able to provide all these by default. With multiple Database and schemas in picture, it wont be the same. So you revisit the whole code to understand the patterns of what tables are accessed together, read or written together and group then accordingly. Any place where it is not according to the newly designed schema, either you have to change code or design or come up with hacks. This exercise has been more painful than you thought. But it also validated your domain separation ideas. While majority of the data belonging to a domain were rightly placed into corresponding schemas, there are few schemas/tables which are being shared across domains.  
  
**Read/Write services and caching**  
  
You have decided that direct db access at multiple places are causing issues. Few critical DB/schema in general has become quite hot and even after split there is a lot of load. You have started to investigate if reads and writes could be separated and served from two different instances. You also realized that doing that may result into issues for the scenarios where replication delays may cause confusion/issues to the customers. You zeroed in on a in memory caching solution. All the writes would happen to the db and to the cache as well at the same time. The read will happen through cache first and if required additionally from read only DB instance for partial use cases. If the data is not available in cache, then only it will go to the read only db instance. This also means direct db access from different places is not a good idea and all the database reads and writes should happen through services which know the logic for when to look up from cache vs when to look at the db.
  
With this change, database health looks good. Read and writes are separate and caching is helping a lot with the reduction of read query hits to database.
  
Your app continues to attaract more customers and exponential traffic. With the introduction of batches and customer facing apis, the load at peak is several times the load on an average.
  
**Delayed write, journaling and eventual consistency**  
  
The DB look fragile during the peak traffic. If somehow you can distribute the peak load to off peak hours, you should be good. You are trying to figure out how do you delay the write in case of peak load. You come up with a very trivial solution of a queue and a daemon. You always write to a temporary database and later a daemon picks up the data from this temporary database and writes into main data base. Before writing into the temporary database, you make sure the data is good by validating data, application logic based on the data also update the cache incase someone needs the data immediately. You also understand that for temporary database tables journaling will be a good idea; incase any rollback is required. So, you also made sure that the temporary database tables are insert, very similar to append only logs, where you never update a row and always keeps adding new records for each operation. This way it helps you reconcile that records and also provide a way to do a rollback by doing the semantically inverse operations in case of a rollback scenario.Now instead of one main database which was a chocking point, you are going with multiple temporary database tables, so the bottleneck is removed. Daemon always picks at a defined speed and writes to the main db at a same rate. In case of a spike, all the load is still limited to temporary dbs and main db writes are still at the same pace. You understand that during peak traffic, the actual write to main db could be delayed a bit. To take care of reads in the meantime, you have already written to cache. Your read service assumes that if cache has the data, it will be the most up-to-date data, so read from the cache else read from the database.
  
This solves your peak traffic issues. Your code needed a lot more data validation though and many of these validation were not explicit and you really had to put lot of break and fix kind of patches to make this work. Once it’s done, you are a relieved man. Main database can, now, be maintained more regularly without the fear of dropping traffic.
  
But the story doesn’t end here.  
  
Your app’s growth is unstoppable!
  
It’s continuing to grow at enormous pace and it seems the number of writes per second into the main tables for key schemas are well beyond the average recommended numbers. Write instances are running hot again and you need to do something.  
  
**Database sharding**  
  
Each db can take only a finite load and even after upgrading the hardwares, you don’t see it can fit the scale you need. You do some research and find the only way out is having multiple dbs each serving a fraction of work load. You like this idea of sharding. You have just finished your analysis and it seems you have found that you need to have one sharding key which will decide which target database the data needs to be written to or read from. You already have read and write services for accessing database so it would be a bit simpler for you. You have analyzed all your read and write patterns and figured out that you can shard the db based on Account Number. You are going to make sure all the read queries have account number as one of the key. Similarly every write must have an account number. You are also going to have a separation between physical shards and logical shards, so that you can start with less number of physical databases but eventually scale out to more databases as and when need arises.
  
With this you have kind of reached to the limit of maximum scale that probably your app can have.
  
If you need further scale, probably a split of your company may be the next option!  
  
## References
  
[How To Set Up Nginx Load Balancing](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-load-balancing)  
[Microservices need to honor contracts](https://www.infoworld.com/article/3198149/microservices-need-to-honor-contracts.html)  
[Microservices: Handling eventual consistency](https://softwareengineering.stackexchange.com/questions/354911/microservices-handling-eventual-consistency)  
[Eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency)  
[Database sharding explained in plain English](https://www.citusdata.com/blog/2018/01/10/sharding-in-plain-english)  
