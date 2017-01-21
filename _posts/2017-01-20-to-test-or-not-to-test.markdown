---
layout: post
title:  "To Test or Not To Test?"
date:   2017-01-18 00:00:00 -0700
categories: testing
---

When first starting out with unit testing, many developers have the tendency to write tests for anything and everything.
This can be a good learning exercise, but one of the hardest things about unit testing or testing in general 
is not implementing the tests, but determining what is a valuable test. Good tests can catch regressions in your
code before those changes ever make it out of development and save you a lot of headaches. Bad tests, on the other
hand, can be just another maintenance item, costing you time without ever catching problems in your code. 
When you're constrained on time, prioritizing more useful tests over less useful tests is essential.

### What NOT to Test

It's easier to say what not do to than what to do. Some common examples of tests that provide little value 
or should be avoided all together...

#### Property Getters/Setters

It's tempting to test these since they are an easy way to improve your coverage numbers, but unless you're accessors
are doing something extra like validating the input, there's not much value in testing these.

{% highlight php startinline %}
public function getThing() 
{
    return $this->thing;
}

public function setThing($thing)
{
    $this->thing = $thing;
}
{% endhighlight %}

Any test for these methods will likely never break, and if it does it's because the implementation changed and the test
needs to be updated anyway. 

#### Language Features and Third-party Libraries

For example, testing that a call to a type-hinted method fails if passed the wrong type is really just testing that PHP
works. Testing methods of third party libraries should be avoided as well. Any library you use should have its own
tests, and if it doesn't, you may want to use a different library.

#### Tests That Can Never Fail, or Falsifying the Results

This is somewhat common and can be tricky to spot. Sometimes, especially when using mocks, you can end up testing 
artifical results that you create within your test, thus the test could never fail. An simplified example of this, 
when a model returns the result of a stubbed database query.

{% highlight php startinline %}
public function testFetchRowShouldReturnExpectedRow()
{
    $expectedRow = [1, 2, 3];
    
    $dbMock = $this->getMockBuilder(DbAdapter::class)
        ->setMethods('fetchRow')
        ->getMock();
        
    $model = new Model($dbMock);        
    
    // Stubbed out database query to return the thing we're expecting
    $dbMock->method('fetchRow')
        ->willReturn($expectedRow);
    
    // Of course it does, how could it not?
    $this->assertEquals($expectedRow, $model->fetch()); 
}
{% endhighlight %}

### What to Test

So what should we be testing? When making changes to your application, there is always a chance of creating 
regressions, possibly in other parts of the application you didn't think to check. Tests that can catch these 
regressions early will make your life easier in the long run.

#### Requirements

This seems obvious, but if you already have clearly defined requirements, you already know what tests to write.
A good test suite is proof that your application always does what it's supposed to. Any failure in these tests
would almost surely be a bug if it went to production. This often goes outside the scope of unit testing and into
behavioral or end-to-end testing.

#### Sad Path

Another set of tests that practically write themselves. Any time you are throwing and exception, or handling an
expected exception, write a test for it. Sometimes things go wrong. Does your application behave like you would expect
if your database goes down? What happens if you give a method bad input? 

#### Known Bugs

Whenever you fix an existing bug, write a test for the conditions that caused it and the expected behaviot, 
and hopefully you will never see that bug again.

### Why Do We Test?

Testing is not a nice-to-have or a box to check for making good software. A good suite of tests can save you time
and money. It proves your application works. It catches regressions that could cost your customers money, and 
could cost you customers. It gives you the confidence in your application to release code more often, and invest
less time in manual regression testing, and even automate your deployment process.

Robust testing enables you to improve your entire development process. Time spent writing tests is trivial compared
to the time you would spend on bugs, hotfixes, regression testing, and error-prone manual deployments. Life's too
short to not write tests.



