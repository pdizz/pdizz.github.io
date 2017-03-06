---
layout: post
title:  "Behavioral API testing with Behat"
date:   2017-02-07 00:00:00 -0700
categories: testing
---

Recently I've been tasked with refactoring and modernizing some legacy applications, and I was looking 
for a way to ensure that any changes would not break the functionality of existing APIs. Also, given the
complete lack of any documentation for those APIs, I decided to implement a suite of "behavioral" API tests.

### Behat to the Rescue

Behat is the PHP implementation of the [Cucumber] project for Behavioral Driven Development. It uses the Gherkin
syntax to define functionality in the form of user stories and scenarios.
I settled on behavioral testing and Behat for several reasons:

1. Test scenarios are written in human-readable syntax so they can be understood by other stakeholders
1. Once supporting code is in place, additional test scenarios can be written by anyone, not just developers
1. Test scenarios also serve as documentation
1. The application itself was written in PHP

composer require behat/behat
composer require guzzlehttp/guzzle
composer require phpunit/phpunit
vendor/bin/behat --init

{% highlight cucumber %}
Feature: pdizz.github.io site
    As a developer
    I need a blog
    so I can tell people about stuff

    Scenario: I visit the home page
        When I request "http://pdizz.github.io/"
        Then I should get a "200" response
{% endhighlight %}
        
{% highlight php startinline %} 
<?php

use Behat\Behat\Context\Context;
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
     * @When /^I request "(.+)"$/
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
     * @Then /^I should get a "(.*)" response$/
     */
    public function iShouldGetAResponse($httpStatusCode)
    {
        Assert::assertNotNull(
            $this->response,
            'Request did not receive any response, unable to get status code.'
        );

        $code = $this->response->getStatusCode();
        Assert::assertEquals(
            $httpStatusCode,
            $code,
            "Unexpected response code: $code"
        );
    }
}
{% endhighlight %}
        
        
        
    /**
     * @When /^I request "(.+) (.+)"$/
     */
    public function iRequest($verb, $url)
    {
        try {
            $this->response = $this->httpClient->request($verb, $url);
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            $this->request = $e->getRequest();
            if ($e->hasResponse()) {
                $this->response = $e->getResponse();
            }
        }
    }
    
    
    


    Scenario: I view the Behat article
        When I request "http://pdizz.github.io/2017/02/07/behavioral-api-testing-with-behat.html"
        Then I should not get a "404" response
        
        
        
        
        
        

behat.yml

{% highlight yaml %}

default:
  suites:
    default:
      contexts:
        - FeatureContext:
            baseUrl: http://pdizz.github.io

{% endhighlight %}

