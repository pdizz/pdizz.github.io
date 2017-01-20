---
layout: post
title:  "To Test or Not To Test"
date:   2017-01-18 00:00:00 -0700
categories: testing
---

## What should I test?

When first starting out with unit testing, many developers have the tendency to write tests for anything and everything.
This can be a valuable exercise in learning, but one of the hardest things about unit testing or testing in general 
is not implementing the tests, but determining what is a valuable test. Good tests can catch regressions in your
code before those changes ever make it out of development and save you a lot of time and headaches. Bad tests, on the
other hand, can end up just as another maintenance item, costing you time without ever catching problems in your code. 
Especially when you're constrained on time, prioritizing more useful tests over less useful tests is essential.

### What NOT to test

It's usually easier to say what not do to than what to do. Some common examples of tests that provide little value 
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
works. Testing methods of third party libraries should be avoided as well. The libraries you use should have their own
tests, and if they don't, you may want to use a different library.

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


