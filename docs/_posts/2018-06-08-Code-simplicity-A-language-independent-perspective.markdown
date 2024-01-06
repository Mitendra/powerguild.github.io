---
layout: post
title:  "Code simplicity: A language independent perspective"
date:   2018-06-08 08:43:56 -0800
categories: medium blogs

---  
  
![](/assets/images/straight_line.png)  
  
Simple is beautiful is the golden mantra in programming. While efficiency and performance are major factors, simplicity and maintenance cost wins over them in many use cases. These become a deciding factor while choosing a programming language, exploring features in a language or even deciding on standard coding practices within an organization. There are often debates on whether a specific programming language makes it easier to write simpler and maintainable code; quite often projects drift away from a more efficient and performant languages to other languages as the code base starts to grow and number of people working on code base starts to increase and so on.  
  
Many such decisions work really well for some teams, and many of them fail to change the ground realities. Let’s explore, what makes a code simple and what we can do in that regard.  
  
### What makes the code simple  
  
Ironically, writing simple code is neither easy nor simple and, at times, it may actually be quite complex to simplify a logic or a piece of code.  
  
A few signs of simple and easy writing:  
  
#### Easy to read and comprehend  
Typically the code is written once and read hundreds of time over a period of time. So, it’s important that this should be easy to read and comprehend. 
   
Some simple tips that can help:  
1. Avoid double negations: instead of saying if(! not_present) use if(present).  
      
2. Avoid misleading abbreviations : Once we were working on a project which was related to Point of Sale. Since it was a bit longer, we shortened it to POS. To check whether a specific object belongs to POS or not, we added a method called isPOS that would take an object and return true or false accordingly. The object also has a notion of positive or negative(it was somewhat related to amount as well). A new developer who had recently joined the team, had to add some feature and was needed to check if the object has positive amount, he used this method isPOS(assuming its a shortcut for isPositive) and continued. Since most of the amount was supposed to be negative and also very few objects was supposed to be related to POS, majority of his tests passed correctly. No one caught this in code review as well and was caught very late in the cycle, after the code went to production.  
   
3. Go with natural flow for naming conventions : If variable w is used for ResponseWriter, r would typically mean ResponseReader, don’t use it for Request.  
   
4. Avoid multiple complex conditions in a if blocks : When working on a legacy code, it’s too tempting to just add one more condition and get the feature done. While most of the time it works great, these extra conditions keeps on changing the business intent and after few iterations the business logic becomes completely unmanageable. So, it’s important whenever you are adding a new condition, make sure it is still relevant in the older context and if not then a little refactoring may be a better option.  
   
5. Avoid magical logic : Efficiency at the cost of readability should be avoided. e.g. X << 3(left shift a number X by 3) instead of X * 8(multiplying X by 8) may be faster but may not have the same amount of readability. Avoiding such stuff typically helps in readability unless the intent is to the same.  
     
#### Easy to comprehend the code/implementation context
Very often people move around and new folks have to look at the code and they may not have the same level of exposure and context of the problem being solved. So, the easier it’s to get the context from the code, the more productive new folks can be. Few things can help:  
  
1. Less state and dependency among methods: methods which are dependent on other methods for state or need to be called in a certain sequences make it harder to read and modify.We have to not only read and understand the current method but also need to understand the sequence of calls. Often this leads to calling a method where the current state is not properly updated, resulting into issues. Do not use member variables in classes to store temporary state to avoid extra arguments in private methods.  
   
2. Specific input and output: many times it’s easier to pass a larger generic data structure to a method, which makes it easier for a later modification, if method needs more input. But, this also typically leads to less reusability of methods. It, also, makes it harder to guess what might be happening on the other side by reading code at the caller side. This forces us to read more code to understand the logic. Similarly, returning a larger structure and filling only few of fields, forces us to read more code to understand what part of the structure could be used after the response, often resulting into accessing data that’s not populated and making code brittle. it’s better to pass only the required input and return a specific output.  
   
3. Less redirections: Doing too many redirections forces us to read much larger code base. Prefer lesser redirections. E.g. in the below code(a bit exaggerated):  

[too_many_redirect.rb](https://gist.github.com/Mitendra/69029757a42447ed7f5e0468a79ad9cd)  
    
```
class URLHelper
	def initialize
		@seperator = '/'
		@domainSeperator = '.'
	end

	def getSchema()
		"https"
	end

	def getSubDomainName()
		"www"
	end
	def getVipHostName()
		"samplevip"
	end

	def getRegion()
		"midwest"
	end

	def getTopLevelDomainName()
		"mycompany.com"
	end
	def getVipName()
		getSubDomainName() + @domainSeperator + getVipHostName() + @domainSeperator + getRegion() + @domainSeperator +  getTopLevelDomainName() 
	end

	def getPrefix(usage)
		if usage == "option1"
			return "option1prefix"
		else
			return "defaultprefix"
		end
	end
	def getBasePath(usage)
		getSchema() + @seperator + @seperator + getVipName() + @seperator + getPrefix(usage)
	end

	def getSuffix(usage)
		"defaultSuffix"
	end
	
	def getCommonPath(usage)
		getBasePath(usage) + @seperator + getSuffix(usage)
	end

	def getBaseUrl(usage)
		getCommonPath(usage) + "/SpecificValue"
	end

end

url = URLHelper.new.getBaseUrl("option1")
puts url
# https//www.samplevip.midwest.mycompany.com/option1prefix/defaultSuffix/SpecificValue
```

There are lot of stuffs which are most probably not going to change. This kind of unnecessary abstraction makes it harder to get the context.
So it might be worthy to just replace all of these with one simple method:  

[simplified_redirects.rb](https://gist.github.com/Mitendra/69029757a42447ed7f5e0468a79ad9cd)  
   
```
class URLHelper
  def getBaseUrl(usage)
    if(usage == "option1")
      return "https://www.samplevip.midwest.mycompany.com/option1prefix/defaultSuffix/SpecificValue"
    else
      return "https://www.samplevip.midwest.mycompany.com/defaultprefix/defaultSuffix/SpecificValue"
    end
  end
end
```

#### Easy to get the intent and the problem context  
when it’s time to modify the existing code, the context of the problem and the intent help more than the implementation steps. So a meaningful abstraction is important.  
   
**Meaningful abstractions**  
     
We can write a very specific code that works perfectly fine for a context but may hide the intent.  
e.g. Lets say, we are building an e-commerce site and once the checkout is complete, we need to send an email to indicate checkout is complete and if there are errors we may need to send a different email. Assume there are 2 steps before checkout is considered completed and we can have scenarios where after 1st step, the system errors out. Also assume we record success of each step as a row in DB(in memory or actual DB).  
Here is one sample implementation.  
  
[missing_intend.rb](https://gist.github.com/Mitendra/b9cb86b48ded20f8c52c926450039954#file-missing_intend-rb)  
  
```
def sendEmail(numOfRows)
  if(numOfRows  == 1)
    # send an email saying, checkout errored out
  else
    # send an email saying, checkout is complete
  end
end
```
Although this may work perfectly fine for a given condition, this doesn’t tell the intend correctly and focuses more on lot of current assumptions. If later we add a new step, this code will break. Modifying this code for a new person will be super hard as the intent of the specific steps are missing.  
  
In these cases, an implementation with more code but with right intent may be better:  
  
[using_intend.rb](https://gist.github.com/Mitendra/aa42393a5b419a38e0cf23460a1afef9)  

```
def sendEmail(checkoutDetails)
  if isCheckoutCompleted(checkoutDetails)
    # send an email saying checkout is complete
  else
    # send an email saying checkout errored out
  end
end

def isCheckoutCompleted(checkoutDetails)
  if(checkoutDetails.length == 1) # same as no of rows
    return true
  else
    return false
  end
end
```

### Simple is beautiful
At the end what matters is to convey the right context and meaning. A code which is simple to read and scan through but not revealing the right intent may not be treated as simple. While languages and its feature may play an important role, if we take care of these basic things, chances are our code will still be beautiful, irrespective of what language, what paradigm or what features of a language we are using.  
