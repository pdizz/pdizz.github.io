---
layout: post
title:  "Testing REST API's"
date:   2017-04-07 0:00:00 -0700
categories: testing
series: behat-api
order: 3
---

### HTTP Methods in the Request

Currently our `I request ""` method only allows us to make GET requests. For testing REST API's we need to be able
to specify different HTTP verbs for the request. Change the annotation for `iRequest()` in `features/bootstrap/FeatureContext.php` 
to capture another parameter for the http verb and use it in the method.

{% highlight php startinline %}
    /**
     * @When /^I request "(.+) (.+)"$/
     */
    public function iRequest($verb, $route)
    {
        try {
            $this->response = $this->httpClient->request($verb, $route);
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            $this->request = $e->getRequest();
            if ($e->hasResponse()) {
                $this->response = $e->getResponse();
            }
        }
    }
{% endhighlight %}

Now we can specify the http verb along with the route in our test steps...

{% highlight text %}
Feature: pdizz.github.io site
    As a developer
    I need a blog
    so I can tell people about stuff

    Scenario: I visit the home page
        When I request "GET /"
        Then I should get a "200" response

    Scenario: I view the Behat article
        When I request "GET /testing/2017/03/22/behavioral-api-testing-with-behat-01.html"
        Then I should get a "200" response
{% endhighlight %}

### Getting Data

To facilitate these tests I created a mock API using [npm json-server](https://github.com/typicode/json-server) 
with the route `/client`. Check out the behat project for this series to test it out 
[github.com/pdizz/pdizz_behat](https://github.com/pdizz/pdizz_behat)

First lets modify the `iRequest()` method to store the response body so we can inspect the results of our API requests.

{% highlight php startinline %}    
    /** @var string */
    protected $responseBody;
    
    /**
     * @When /^I request "(.+) (.+)"$/
     */
    public function iRequest($verb, $route)
    {
        try {
            $this->response = $this->httpClient->request($verb, $route);
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            $this->request = $e->getRequest();
            if ($e->hasResponse()) {
                $this->response = $e->getResponse();
            }
        } finally {
            if (isset($this->response)) {
                $this->responseBody = $this->response->getBody()->getContents();
            }
        }
    }
{% endhighlight %}

Now we can create some more useful step definitions to help us inspect the JSON response body from the API. 
Create a new feature file `features/client-api.feature`

{% highlight text %}
# features/client-api.feature
Feature: Client REST API
    As a guy who does business stuff
    I need a client API
    so I can keep track of my many clients

    Scenario: I get my first client
        When I request "GET /client/1"
        Then I should get a "200" response
        And the response should contain ""id": 1"
        And the response "firstName" field should be "John"
        And the response "lastName" field should be "Doe"
        And the response "address" field should exist
        And the response "address.state" field should be "CO"
{% endhighlight %}

And the methods in `FeatureContext` to support these steps. (This assumes the API is serving JSON data, but these can 
be easily modified to handle XML and other data types as well).

{% highlight php startinline %}
    /**
     * @Then /^the response should contain "(.*)"$/
     */
    public function theResponseShouldContain($string)
    {
        Assert::assertContains($string, $this->responseBody);
    }

    /**
     * @Given /^the "(.*)" field should exist$/
     */
    public function theFieldShouldExist($fieldPath)
    {
        $body = json_decode($this->responseBody, true);
        $fields = explode('.', $fieldPath);

        $path = [];

        while (count($fields) != 0) {
            $field = array_shift($fields);
            $path[] = $field;
            Assert::assertArrayHasKey(
                $field,
                $body,
                "Unable to find field " . join('.', $path) . PHP_EOL . $this->responseBody
            );

            $body = $body[$field];
        }

        return $body;
    }

    /**
     * @Given /^the "(.*)" field should be "(.*)"$/
     */
    public function theFieldShouldBe($fieldPath, $value)
    {
        $body = $this->theFieldShouldExist($fieldPath);

        Assert::assertEquals(
            $value,
            $body,
            "The $$fieldPath field did not contain the expected value $value. It was " . $body . PHP_EOL . $this->responseBody
        );
    }
{% endhighlight %}

### Sending Data

Testing the response is great and all but we'll need a way to send data as well. We can do this by creating more steps
to build up our request before sending it. The scenario looks something like this:

{% highlight text %}
    Scenario: I add my second client
        Given a request body "{"id":2,"firstName":"Jane","lastName":"Doe","address":{"street":"123 Main St","city":"Denver","state":"CO"}}"
        When I request "POST /client"
        Then the "id" field should be "2"
        And the "firstName" field should be "Jane"
        And the "lastName" field should be "Doe"
{% endhighlight %}

Now in the FeatureContext we'll need a method for "Given a request body" which sets the request body prior to making the
request, then modify `iRequest()` to add it to the request options...

{% highlight php startinline %}
    /**
     * @Given /^a request body "(.*)"$/
     */
    public function aRequestBody($string)
    {
        $this->requestBody = (string) $string;
    }
    
    /**
     * @When /^I request "(.+) (.+)"$/
     */
    public function iRequest($verb, $route)
    {
        $options['body']    = $this->requestBody;
        $options['headers'] = ['Content-Type' => 'application/json'];

        try {
            $this->response = $this->httpClient->request($verb, $route, $options);
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            $this->request = $e->getRequest();
            if ($e->hasResponse()) {
                $this->response = $e->getResponse();
            }
        } finally {
            if (isset($this->response)) {
                $this->responseBody = $this->response->getBody()->getContents();
            }
        }
    }
{% endhighlight %}

If you prefer multi-line strings you can take advantage of the included `Behat\Gherkin\Node\PyStringNode` to do
python-style strings in your scenarios. Here's an example using PUT to update the client data making use of a 
multi-line string for the request body.

{% highlight text %}
    Scenario: I fix a typo in my client's name
        Given a request body with:
        """
        {
            "firstName":"Jayne",
            "lastName":"Doe",
            "address": {
                "street":"123 Main St",
                "city":"Denver",
                "state":"CO"
            }
        }
        """
        When I request "PUT /client/2"
        Then the "id" field should be "2"
        And the "firstName" field should be "Jayne"
        And the "lastName" field should be "Doe"
{% endhighlight %}

{% highlight php startinline %}
    /**
     * @Given /^a request body with:$/
     */
    public function aRequestBodyWith(PyStringNode $string)
    {
        $this->requestBody = (string) $string;
    }
{% endhighlight %}

Finally we need to consider what happens if we run this test again. Now that the client has been created, 
the test will fail every time it's run after the first time. One of the biggest issues with testing live API's is 
managing the data, and that topic could take a whole series to cover, but one of the most basic ways of handling this is
just cleaning up after ourselves. I'll create one more scenario to delete the client so we'll be starting from the
same place each time this scenario is run.

{% highlight text %}
    Scenario: I delete my second client
        When I request "DELETE /client/2"
        Then I should get a "200" response
{% endhighlight %}

This isn't necessarily the most reliable way since we will rarely be able to count on "id" being the same every time,
or knowing what it will be in advance. In the next article we'll look at some ways to deal with this, like how to use 
variables in our scenarios instead of hard-coded values, and how to capture data from a response to use in other steps.
