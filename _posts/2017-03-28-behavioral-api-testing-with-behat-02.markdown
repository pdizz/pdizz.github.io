---
layout: post
title:  "Running Behat in Different Environments"
date:   2017-03-28 0:00:00 -0700
categories: testing
series: behat-api
order: 2
---

### Profiles

Before long we're going to want to run our tests against different environments. To accomplish this we can configure 
profiles for our test suite in `behat.yml`. (Check out the behat repo to follow along 
[github.com/pdizz/pdizz_behat](https://github.com/pdizz/pdizz_behat))

{% highlight yaml %}
# behat.yml
default:
  suites:
    default:
      contexts:
        - FeatureContext:
            baseUrl: http://127.0.0.1:4000

production:
  suites:
    default:
      contexts:
        - FeatureContext:
            baseUrl: http://pdizz.github.io
{% endhighlight %}

This defines a production profile, and adds context parameters for the default profile and the new production profile.
These context parameters will be passed in to the constructor of `FeatureContext` so modify 
`features/bootstrap/FeatureContext.php` to set $baseUrl in the constructor and use it in the request. 

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

    /** @var string */
    protected $baseUrl;

    /**
     * Initializes context.
     *
     * Every scenario gets its own context instance.
     * You can also pass arbitrary arguments to the
     * context constructor through behat.yml.
     */
    public function __construct($baseUrl)
    {
        $this->baseUrl = $baseUrl;
        $this->httpClient = new GuzzleHttp\Client([
            'base_uri' => $baseUrl
        ]);
    }

    /**
     * @When /^I request "(.+)"$/
     */
    public function iRequest($route)
    {
        try {
            $this->response = $this->httpClient->get($route);
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

Now we can remove the hard-coded base url from our test steps and just specify the route...

{% highlight text %}
Feature: pdizz.github.io site
    As a developer
    I need a blog
    so I can tell people about stuff

    Scenario: I visit the home page
        When I request "/"
        Then I should get a "200" response

    Scenario: I view the Behat article
        When I request "/testing/2017/03/22/behavioral-api-testing-with-behat-01.html"
        Then I should get a "200" response
{% endhighlight %}

Now running `vendor/bin/behat` will run the test suite against the development environment at `http://127.0.0.1:4000`
since that is the default profile. To run the same tests against the production profile we can run 
`vendor/bin/behat -p production`
