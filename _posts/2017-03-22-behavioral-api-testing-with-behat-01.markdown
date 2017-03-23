---
layout: post
title:  "Behavioral API testing with Behat"
date:   2017-03-22 0:00:00 -0700
categories: testing
---

Recently I've been tasked with refactoring and modernizing some legacy applications, and I was looking 
for a way to ensure that any changes would not break the functionality of existing APIs. Also, given the
complete lack of any documentation for those APIs, I decided to implement a suite of "behavioral" API tests.

### Behat to the Rescue

Behat is the PHP implementation of the [Cucumber](https://en.wikipedia.org/wiki/Cucumber_(software)) project for 
Behavioral Driven Development. It uses the Gherkin syntax to define functionality in the form of user stories 
and scenarios. I settled on behavioral testing and Behat for several reasons:

1. Test scenarios are written in human-readable syntax so they can be understood by other stakeholders
1. Once supporting code is in place, additional test scenarios can be written by anyone, not just developers
1. Test scenarios also serve as documentation
1. The application itself was written in PHP

### Getting Started

Create a new project and install some libraries. We'll need behat itself, an assertion library (I've chosen phpunit)
and an http client to make calls to the API.

{% highlight bash %}
composer require behat/behat phpunit/phpunit guzzlehttp/guzzle
{% endhighlight %}

Behat tests are written using the human-readable [Gherkin](https://en.wikipedia.org/wiki/Cucumber_(software)#Gherkin_.28Language.29)
language in "feature" files where you define scenarios that describe how the feature will be used. Scenarios are
composed of a number of sentences that are the testing steps for that scenario.

When Behat runs a feature file it loads a php class called a feature context, and matches each sentence
to a method in the feature context which has the php code to support that test step.

Initialize a new behat project in your project directory. This will create an empty file, 
features/bootstrap/FeatureContext.php, that we'll use to support our test steps.

{% highlight bash %}
vendor/bin/behat --init
{% endhighlight %}

### A Basic Feature

Now we're ready to write the first feature...

{% highlight text %}
# features/pdizz.github.io.feature
Feature: pdizz.github.io site
    As a developer
    I need a blog
    so I can tell people about stuff

    Scenario: I visit the home page
        When I request the home page
        Then I should get a successful response
{% endhighlight %}

Now for the code. We can use the __construct() method to set up our http client. Then we need methods to support our 
first two sentences "When I request the home page" and "Then I should get a successful response". Behat uses regex
to match the sentence in the feature file with annotations in the FeatureContext docblocks. Our FeatureContext.php
should look something like this:
        
{% highlight php startinline %} 
<?php

use Behat\Behat\Context\Context;
use Behat\Gherkin\Node\PyStringNode;
use Behat\Gherkin\Node\TableNode;
use PHPUnit\Framework\Assert;

/**
 * Defines application features from the specific context.
 */
class FeatureContext implements Context
{
    protected $httpClient;

    /** @var \GuzzleHttp\Psr7\Request */
    protected $request;

    /** @var \GuzzleHttp\Psr7\Response */
    protected $response;

    /**
     * Initializes context.
     *
     * Every scenario gets its own context instance.
     * You can also pass arbitrary arguments to the
     * context constructor through behat.yml.
     */
    public function __construct()
    {
        $this->httpClient = new GuzzleHttp\Client();
    }

    /**
     * @Given /^I request the home page$/
     */
    public function iRequestTheHomePage()
    {
        try {
            $this->response = $this->httpClient->get('http://pdizz.github.io/');
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            $this->request = $e->getRequest();
            if ($e->hasResponse()) {
                $this->response = $e->getResponse();
            }
        }
    }

    /**
     * @Then /^I should get a successful response$/
     */
    public function iShouldGetASuccessfulResponse()
    {
        Assert::assertNotNull(
            $this->response,
            'Request did not receive any response, unable to get status code.'
        );

        $code = $this->response->getStatusCode();
        Assert::assertEquals(200, $code, "Unexpected response code: $code");
    }
}
{% endhighlight %}
    
Notice the annotations in the docblocks for `iRequestTheHomePage()` and `iShouldGetASuccessfulResponse()`?
Annotations should start with one of the keywords `@Given`, `@When` or `@Then` and step definitions should
start with `Given`, `When`, `Then`, `And`, or `But`. These words are interchangeable for readability, so the 
sentence "When I request the home page" will still match up with `@Given /^I request the home page$/`

Let's try running the feature

{% highlight bash %}
pete@pete-laptop:~/Projects/pdizz_behat$ vendor/bin/behat
Feature: pdizz.github.io site
    As a developer
    I need a blog
    so I can tell people about stuff

  Scenario: I visit the home page           # features/pdizz.github.io.feature:6
    When I request the home page            # FeatureContext::iRequestTheHomePage()
    Then I should get a successful response # FeatureContext::iShouldGetASuccessfulResponse()

1 scenario (1 passed)
2 steps (2 passed)
0m0.11s (9.70Mb)
{% endhighlight %}

### Parameters in Step Definitions

We have a couple steps defined, but they are not very re-usable in their current state. We can add parameters to our
step definitions to make them much more useful. Let's modify `iRequestTheHomePage()` first. We need to change the 
annotation to use regex to capture the parts of the sentence we want to be parameterized, the url in this case, 
and change the method signature to include the $url parameter.
 
Likewise we can change `iShouldGetASuccessfulResponse()` to accept the expected http status code as a paramter.

{% highlight text %}
    Scenario: I visit the home page
        When I request "http://pdizz.github.io/"
        Then I should get a "200" response
{% endhighlight %}

{% highlight php startinline %}
<?php
...
    /**
     * @Given /^I request "(.+)"$/
     */
    public function iRequest($url)
    {
        try {
            $this->response = $this->httpClient->get($url);
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            $this->request = $e->getRequest();
            if ($e->hasResponse()) {
                $this->response = $e->getResponse();
            }
        }
    }

    /**
     * @Then /^I should get a "(.+)" response$/
     */
    public function iShouldGetAResponse($expectedCode)
    {
        Assert::assertNotNull(
            $this->response,
            'Request did not receive any response, unable to get status code.'
        );

        $actualCode = $this->response->getStatusCode();
        Assert::assertEquals(
            $expectedCode,
            $actualCode,
            "Unexpected response code: $actualCode"
        );
    }
}
{% endhighlight %}

Now our step definitions can be used to write other scenarios. Let's try it. I'll write a scenario to cover this
article using the step definitions we already have:

{% highlight text %}
    Scenario: I view the Behat article
        When I request "http://pdizz.github.io/testing/2017/03/22/behavioral-api-testing-with-behat-01.html"
        Then I should get a "200" response
{% endhighlight %}

In the next article we'll expand on our step definitions and see some more techniques useful in testing different
types of API's.
