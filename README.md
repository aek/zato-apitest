zato-apitest - API Testing for Humans
=====================================

Introduction
------------

zato-apitest is a friendly command line tool for creating beautiful tests of HTTP-based REST, XML and SOAP APIs with as little
hassle as possible.

Tests are written in plain English, with no programming needed, and can be trivially easy extended in Python if need be.

Note that zato-apitest is meant to test APIs only. It's doesn't simulate a browser nor any sort of user interactions. It's meant
purely for machine-machine API testing.

Originally part of [Zato] (https://zato.io) - open-source ESB, SOA, REST, APIs and cloud integrations in Python.

In addition to HTTP Zato itself supports AMQP, ZeroMQ, WebSphere MQ, including JMS, Redis, FTP, OpenERP, SMTP, IMAP, SQL, Amazon S3,
OpenStack Swift and more so it's guaranteed zato-apitest will grow support for more protocols and transport layers with time.

Here's how a built-in demo test case looks like:

![zato-apitest demo run](https://raw.githubusercontent.com/zatosource/zato-apitest/master/docs/gfx/demo.png)

What it can do
--------------

- Invoke HTTP APIs

- Use [JSON Pointers] (https://zato.io/blog/posts/json-pointer-rfc-6901.html) or [XPath] (https://en.wikipedia.org/wiki/XPath)
  to set request's elements to strings, integers, floats, lists, random ones from a set of values, random strings, dates now/random/before/after/between.
  
- Check that JSON and XML elements, exist, don't exist,
  that an element is an integer, float, list, empty, non-empty, that it belongs to a list or doesn't.

- Set custom HTTP headers, user agent strings, method and SOAP action.

- Check that HTTP headers are or are not of expected value, that a header exists or not, contains a value or not, is empty or not,
  starts with a value or not and ends with a value or not.
  
- Read configuration from environment and config files.

- Store values extracted out of previous steps for use in subsequent steps, i.e. get a list of objects, pick ID of the first one
  and use this ID in later steps.
  
- Can be integrated with JUnit

- Can be very easily extended in Python

Download and install
--------------------

Newest releases are always available [on PyPI] (https://pypi.python.org/pypi/zato-apitest)
and can be installed with [pip] (https://pip.pypa.io/en/latest/installing.html) after installing a few system prequisites:

* PostgreSQL development libraries
* XML development-related libraries
* Python headers
* YAML headers

For instance, on Debian/Ubuntu:

```bash
$ sudo apt-get install -y libpq-dev libxml2-dev libxslt1-dev python-dev libyaml-dev
```

Now, on to zato-apitest:

```bash
$ sudo pip install zato-apitest
```

Run a demo test
---------------

Having installed the program, running ```apitest demo``` will set up a demo test case, run it against a live environment
and present the results, as on the screenshot in Introduction above.

Note that it may a good idea to check the demo closer, copy it over to a directory of your choice, and customize things to learn
by playing with an actual set of assertions.

Workflow
--------

1. Install zato-apitest
2. Initialize a test environment by running ```apitest init /path/to/an/empty/directory```
3. Update tests
4. Execute ```apitest run /path/to/tests/directory``` when you are done with updates
5. Jump to 3.

Tests and related resources
---------------------------

Let's dissect directories that were created after running ```apitest init```:

```
~/mytests
├── config
│   └── behave.ini
└── features
    ├── demo.feature
    ├── environment.py
    ├── json
    │   ├── request
    │   │   └── demo.json
    │   └── response
    │       └── demo.json
    ├── steps
    │   └── steps.py
    └── xml
        ├── request
        │   └── demo.xml
        └── response
```


 Path                              | Description
---------------------------------- | ----------------------------------------------------------------------------------------------------------------------
```./config/behave.ini```          | Low-level configuration that is passed to the underlying [behave] (https://pypi.python.org/pypi/behave) library as-is.
```./features/demo.feature```      | A set of tests for a single feature under consideration.
```./features/environment.py```    | Place to keep hooks invoked throughout a test case's life-cycle in.
```./features/json/request/*```    | JSON requests, if any, needed as input to APIs under tests.
```./features/json/response/*```   | JSON responses you expect for APIs to produce, used for smart comparison.
```./features/steps/steps.py```    | Custom assertions go here.
```./features/xml/request/*```     | XML requests (including SOAP), if any, needed as input to APIs under tests.
```./features/xml/response/*```    | *(Currently not used, future versions will allow for comparing XML/SOAP responses directly)*

Each set of related tests concerned with a particular feature is kept in its own *.feature file in /features.

For instance, if you test an API allowing one to create and update customers, you could have the following files:

```
└── features
    ├── cust-create.feature
    └── cust-update.feature
```

How you structure the tests is completely up to you as long as individual files end in *.feature.

Anatomy of a test
-----------------

Here's how a sample test kept in ```./features/cust-update.feature``` may look like. For comparison, it shows both SOAP and REST
assertions. This is the literal copy of a test, everything is in plain English:

```
Feature: Customer update

Scenario: SOAP customer update

    Given address "http://example.com"
    Given URL path "/xml/customer"
    Given SOAP action "update:cust"
    Given HTTP method "POST"
    Given format "XML"
    Given namespace prefix "cust" of "http://example.com/cust"
    Given request "cust-update.xml"
    Given XPath "//cust:name" in request is "Maria"
    Given XPath "//cust:last-seen" in request is a random date before "2015-03-17" "default"

    When the URL is invoked

    Then XPath "//cust:action/cust:code" is an integer "0"
    And XPath "//cust:action/cust:msg" is "Ok, updated"

    And context is cleaned up

Scenario: REST customer update

    Given address "http://example.com"
    Given URL path "/json/customer"
    Given query string "?id=123"
    Given HTTP method "PUT"
    Given format "JSON"
    Given header "X-Node" "server-test-19"
    Given request "cust-update.json"
    Given JSON Pointer "/name" in request is "Maria"
    Given JSON Pointer "/last-seen" in request is UTC now "default"

    When the URL is invoked

    Then JSON Pointer "/action/code" is an integer "0"
    And JSON Pointer "/action/message" is "Ok, updated"
    And status is "200"
    And header "X-My-Header" is "Cool"
```

- Each test begins with a ```Feature: ``` preamble which denotes what is being tested
- A test may have multiple scenarios - here one scenario has been created for each SOAP and REST
- Each scenario has 3 parts, corresponding to building a request, invoking an URL and running assertions on a response received:

  - One or more ```Given``` steps
  - Exactly one ```When``` step
  - One or more ```Then/And``` steps. There is no difference between how ```Then``` and ```And``` work, simply the first
    assertion is called ```Then``` and the rest of them is ```And```. Any assertion may come first.

- In both ```Given``` and ```Then/And``` the order of steps is always honored.

- Steps work by matching patterns that can be potentially parametrized between double quotation marks,
  for instance ```Given address "http://example.com"``` is an invocation of a ```Given address "{address}"``` pattern.


Available steps and assertions
------------------------------

All the [default steps are listed separately] (./docs/steps/list.md). You're also encouraged to [browse the usage examples] (./docs/steps).


Where to keep configuration
---------------------------

Configuration of the test scenarios can be kept in and read from 3 places:

- Environment variables
- ./features/config.ini
- Test case-specifc context

The rules are:

- Any value prefixed by '$' is read from an environment variable
- Any value prefixed by '@' is read from ./features/config.ini's [user] stanza
- Any value prefixed by '#' is read from the current test case's context

Additionally, please keep in mind that individual tests can store variables basing on previous steps or responses hence combining
all the configuration options allows one to form advanced scenarios, such as the one below.

```
Feature: zato-apitest docs

Scenario: Prepare data

    Given address "$MYAPP_ADDRESS"
    Given URL path "@MYAPP_PATH_LOGIN"
    Given format "JSON"
    Given I store "Maria Garca" under "cust_name"
    Given request is "{}"
    Given JSON Pointer "/customer_id" in request is "$MYAPP_DEFAULT_CUSTOMER"
    
    When the URL is invoked

    Then I store "/login" from response under "cust_login"

Scenario: Get customer payments

    Given address "$MYAPP_ADDRESS"
    Given URL path "@MYAPP_PATH_PAYMENTS"
    Given format "JSON"
    Given request is "{}"
    Given JSON Pointer "/cust_login" in request is "#cust_login"
    Given JSON Pointer "/cust_name" in request is "#cust_name"

    When the URL is invoked

    Then status is "200"
```

Discussion:

- First scenario prepares data needed for the actual test performed by the second one
- MYAPP_ADDRESS is an environment variable that can change from host to host without being hardcoded in test's body
- MYAPP_PATH_LOGIN is a variable stored in ./features/config.ini's [user] stanza
- Variable 'cust_name' is set to a static value of 'Maria Garca'
- Variable 'cust_login' is set to a value returned in response to the fist scenario
- Second scenario makes use of data prepared by the first one
- Remeber to [clean up the context] (./docs/steps/step_then_context_is_cleaned_up.md) if you actually do not want
  for any config variables to be carried over from a step to subsequent ones


Extending zato-apitest and adding custom assertions
---------------------------------------------------

zato-apitest comes with [almost 100 of default steps] (./docs/steps) and it's easy to add new ones.

Let's say that we need to add a step that will return the name of any weekday coming after one provided on input. So, for instance,
if it's Thursday on input, the step should return Friday, Saturday or Sunday.

Each zato-apitest environment contains a ```./features/steps/steps.py``` module. Initially, it only imports all the default steps
but you can simply add your own steps to it. They are always based on [behave] (https://pythonhosted.org/behave/)
so that is where all the additional details are explained.

Here's what needs to be added to ```./features/steps/steps.py``` for the new step to be available:

```python

from random import choice

week_days = 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'

@given('JSON Pointer "{path}" in request is a weekday after "{start}"')
@obtain_values
def given_weekday_after(ctx, path, start):
    """ Returns any weekday after the start one.
    """
    # Sunday is the last day already
    if start == 'Sun':
        raise ValueError('{start} needs to be at most Sat')

    elif start not in week_days:
        raise ValueError('{{start}} ({}) needs to be among {}'.format(
            start, week_days))

    # Build a list of days to pick one from
    start_idx = week_days.index(start)
    remaining = week_days[start_idx+1:]

    # Pick any from the remaining ones after start
    value = choice(remaining)

    # Set it in request
    set_pointer(ctx, path, value)
```

Now you can make use of it in tests, for instance:

```
Feature: zato-apitest docs

Scenario: Extending zato-apitest

    Given address "http://apitest-demo.zato.io"
    Given URL path "/demo/json"
    Given format "JSON"
    Given request is "{}"
    Given JSON Pointer "/day" in request is a weekday after "Fri"

    When the URL is invoked

    Then status is "200"

```

Naming conventions
------------------

The name 'Zato', case insensitive, cannot be used anywhere in your tests. Don't use it as a prefix, suffix or anywhere else. This
applies to step names, variables, functions, anything. This is a system name reserved for the tool's own purposes.

Changelog
---------

* **1.3** - 12-10-2014

  * Fixed installation issues with pip

* **1.2** - 04-10-2014

  * Added SQL
  * Added Cassandra

* **1.1** - 26-06-2014

  * Added HTTP Basic Auth

* **1.0** - 06-06-2014

  * Initial release

License
-------
[LGPLv3] (./LICENSE.txt) - zato-apitest is free to use in open-source and proprietary programs.

