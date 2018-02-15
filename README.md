# Lorikeet

A Parallel test runner for DevOps.

## Overview

Lorikeet is a command line tool and a rust library to run tests for smoke testing and integration testing.  Lorikeet currently supports bash commands and simple http requests along with system information (ram, cpu).

Test plans are defined within a yaml file and can be templated using tera. Each step within a test plan can have multiple dependencies (run some steps before others) and can have expectations about the output of each command.

Steps are run in parallel by default, using the number of threads that are available to your system. If a step has dependencies either by `require` or `required_by` attributes, then it will wait until those steps are finished.

As an example, here's a test plan to check to see whether reddit is up, and then tries to login if it is:

```yaml
check_reddit:
  http: https://www.reddit.com
  regex: the front page of the internet

login_to_reddit:
  http: 
    url: https://www.reddit.com/api/login/{{user}}
    save_cookies: true
    form:
      user: {{user}}
      passwd: {{pass}}
      api_type: json
  jmespath: length(json.errors)
  matches: 0
  require:
    - check_reddit
```

( As a side note, we have added `jmespath: length(json.errors)` & `matches: 0` because an invalid login to reddit still returns a status of `200 OK` )

And the output of lorikeet:

```yaml
$ lorikeet -c config.yml test.yml
- name: check_reddit
  pass: true
  output: the front page of the internet
  duration: 1416.591ms

- name: login_to_reddit
  pass: true
  output: 0
  duration: 1089.0276ms
```

The name comes from the [Rainbow Lorikeet](https://en.wikipedia.org/wiki/Rainbow_lorikeet), an Australian Bird which is very colourful.  Like a canary in a coal mine, lorikeet is meant to provide a way of notifying when things go wrong. Rather than running one test framework (one colour), it is meant to be more full spectrum, hence the choice of a bird with rainbow plumage.

They are also very noisy birds.

## Changes in `0.6.0`

*Breaking Change* Adjusted the step to always include an output, even on failure.  This is reflected in the webhook submission also.  This allows for polling things like memory which may fail the test, but you still want to record the output of the memory

* Added `filters` to steps.  Allows you to filter or change the output of a step.  At the moment both [jmespath](http://jmespath.org/) and regex are supported.

Here's an example:

```yaml
over_9000:
  description: Checks to see if bitcoin is over 9000 USD
  http: http://preev.com/pulse/units:btc+usd/sources:bitstamp
  jmespath: btc.usd.bitstamp.last
  greater_than: 9000
```

## Installation

Lorikeet is on crates.io, so you can either run:

```sh
cargo install lorikeet
```

Or clone and build this repo:

```sh
cargo build --release
```

## Usage

Command line usage is given by `lorikeet -h`:

```
USAGE:
    lorikeet [FLAGS] [OPTIONS] [test_plan]

FLAGS:
    -h, --help       Prints help information
    -q, --quiet      Don't output results to console
    -V, --version    Prints version information

OPTIONS:
    -c, --config <config>      Configuration File
    -w, --webhook <webhook>    Webhook submission URL

ARGS:
    <test_plan>    Test Plan [default: test.yml]
```

### Test Plan

The test plan is the main driver for lorikeet and is already quite flexible.  See below for examples and test syntax.  By default lorikeet will expect a file `test.yml` in the current directory.

### Config Option

Lorikeet uses [tera](https://github.com/Keats/tera) as a template engine so you can include variables within your yaml test plan.  Using `-c` you can provide the context of the test plan as a seperate yaml file.  This file can be in any shape, as long as it's valid yaml.

As an example, say you want to check that a number of servers are up and connected.  You can have a config like so:

```yaml
instances:
  - server1
  - server2
  - server3
```

And then write your test plan:

```yaml
{% for instance in instances %}

ping_server_{{instance}}:
  bash: ping -c 1 {{instance}} 2>&1 >/dev/null

{% endfor %}
```

And run it:

```yaml
$ lorikeet -c config.yml test.yml
- name: ping_server_server1
  pass: true
  duration: 7.859398ms

- name: ping_server_server2
  pass: true
  duration: 7.95139ms

- name: ping_server_server3
  pass: true
  duration: 7.740785ms
```

### Webhook

You can submit your results to a server using a webhook when the test run is finished.  This will POST a json object with the `submitter::WebHook` shape:

```json
{
    "hostname": "example.hostname",
    "has_errors": true,
    "tests": [{
        "name": "Example Webhook",
        "pass": false,
        "output": "Example Output",
        "error": "Example Error",
        "duration": 7.70
    }]
}
```

## Test Plan syntax

The test plan is a yaml file that is divided up into steps:

```
<step_name>:
  <step_type>: <options>
  (<description>: <value>)
  (<expect_type>: <value>)
  (<filter_type>: <list or value>)
  (<dependence_type>: <list or value>)
```

Each step has a unique name and a step type.  Optionally, there can be an expect type, and a list of dependencies or dependents.

You can also include a description of what the test does alongside a name, so you can provide a more detailed explanation of what the test is doing

### Step Types

There are currently 4 step types that can be configured: bash, http, system and value

#### Bash Step type

The bash step type simply runs the `bash` command to execute shell scripts:

```yaml
say_hello:
  bash: echo "hello"
```

Optionally you can specify not to return the output if you're only interested in the return code of the application:

```yaml
dont_say_hello:
  bash:
    cmd: echo "hello"
    get_output: false
```

#### HTTP Step Type

The HTTP step type can execute HTTP commands to web servers using reqwest.  Currently this is a very simple step type but does support status codes and storing cookies per domain.

You can specify just the URL:

```yaml
check_reddit:
  http: https://www.reddit.com
  matches: the front page of the internet
```

Or provide the following options:

* `url`: The URL of the request to submit
* `method`: The HTTP method to use, such as POST, GET, DELETE.  Defaults to `GET`
* `get_output`:  Return the output of the request.  Defaults to `true`
* `save_cookies`:  Save any set cookies on this domain.  Defaults to `false`
* `status`: Check the return status is equal to this value.  Defaults to `200`
* `user`: Username for Basic Auth
* `pass`: Password for Basic Auth
* `form`:  Key/Value pairs for a form POST submission.  If method isn't set, then this will set the method to `POST`

As a more elaborate example:

```yaml
login_to_reddit:
  http: 
    url: https://www.reddit.com/api/login/{{user}}
    save_cookies: true
    form:
      user: {{user}}
      passwd: {{pass}}
      api_type: json
```

### System Step Type

The system step type will return information about the system such as available memory or system load using the sys-info crate.


As an example, to check memory:
```yaml
check_memory:
  description: Checks to see if the available memory is greater than 1gb
  system: mem_available
  greater_than: 1048000
```

The system type has a fixed list of values that returns various system info:

* `load_avg_1m`: The load average over 1 minute
* `load_avg_5m`: The load average over 5 minutes
* `load_avg_15m`: The load average over 15 minutes
* `mem_available`:  The amount of available memory
* `mem_free`:  The amount of free memory
* `mem_total`:  The amount of total memory
* `disk_free`:  The amount of free disk space
* `disk_total`:  The total amount of disk space

Using the `greater_than` or `less_than` expect types means you can set thresholds for environment resources:

```yaml
system_load:
  description: Checks the System Load over the last 15 minutes is below 80%
  system: load_avg15m
  less_than: 1.6
```

#### Value Step Type

The value step type will simply return a value, rather than executing anything.

```yaml
say_hello:
  value: hello
```

### Filter types

You can filter your output either via regex, jmespath, or remove the output completely.   Filters can be provided once off, or as a list, so you can chain filters together:

```yaml
example_step:
  value: some example
  filters:
    - regex: some (.*)
```

You can also shorthand provide a filter on the step like so:

```yaml
example_step:
  value: some example
  regex: some
```

**Note: If the filter can't match against a value, it counts as a test error**

#### Regex Filter

Simply filters out the output of the step based upon the matched value.  

```yaml
say_hello:
  value: hello world!
  regex: (.*) world!
```

You can either add it as a `regex` attribute against the step, or in the filter list:

```yaml
say_hello:
  value: hello world!
  filters:
    - regex: (.*) world!
```

By default it will match and return the entire regex statement (`hello world!), but if you only want to match a certain group, you can do that too:

```yaml
say_hello:
  value: hello world!
  regex: 
    matches: (?P<greeting>.*) world!
    group: greeting
```

This will output simply `hello`

#### JMES Path filter

You can use [jmespath](http://jmespath.org/) to filter out JSON documents, returning some or more values:

```yaml
show_status:
  value: "{\"status\": \"ok\"}"
  jmespath: status
```

As with regex, this can be part of a filter chain:

```yaml
show_status:
  value: "{\"status\": \"ok\"}"
  filters:
    - jmespath: status
```

#### No Output Filter

If you don't want your output printed in results, you can add no output:

```yaml
dont_show_hello:
  value: hello
  do_output: false
```

You can also add this to a filter chain:

```yaml
dont_show_hello:
  value: hello
  filters:
    - nooutput
```

Sometimes you might return too much from a request, so you can use this to ensure what's printed out is not included:

```yaml
check_reddit:
  http: https://www.reddit.com
  filters:
    - regex: the front page of the internet

```

### Expect types

There are 3 expect types currently: Match output, Greater than and Less than.  The expect types will take the raw output of the step type and validate against that.  In this way you can use it to match against the returned HTML from a web server, or the output of a bash file.

#### Match Expect type

The match expect type will use regex to match the output of a command.

```yaml
say_hello_or_goodbye:
  value: hello
  matches: hello|goodbye
```

If there is an error converting the regex into a valid regex query, then this will be treated as a failure.

#### Greater than or less than

If your output is numerical, then you can use greater than or less than to compare it:

```yaml
there_are_four_lights:
  value: 4
  less_than: 5
```

### Dependencies

By default tests are run in parallel and submitted to a thread pool for execution.  If a step has a dependency it won't be run until the dependent step has been finished.  If there are no dependencies to a step then it will run as soon as a thread is free.  If you don't specify any dependencies there is no guaranteed ordering to execution.

Dependencies are important when you need to do things like set cookies before checking API, but will cause your tests to take longer to run while they wait for others to finish.

To defined dependencies you can use the `require` and `required_by` arguments to control this dependency tree.  The required steps are given by their name, and can either be a single value or a list of names:

```yaml
step1:
  value: hello

step2:
  value: goodbye
  require: step1

step3:
  value: yes
  require:
    - step1
    - step2
```

Lorikeet will fail to run and panic if:

* There is a circular dependency
* The step name in a dependency can't be found

#### Required By

`required_by` is just the reciprocal of `require` and can be used where the test plan makes it more readable.

So this step plan:

```yaml
step1:
  value: hello

step2:
  value: goodbye
  require: step1
```

Is equivalent to this one:

```yaml
step1:
  value: hello
  required_by: step2

step2:
  value: goodbye
```

#### More complex dependency example

```yaml
you_say_yes:
  value: yes

i_say_no:
  value: no
  require: you_say_yes

you_say_stop:
  value: stop
  require: 
    - i_say_no
    - you_say_yes
  required_by:
    - and_i_say_go_go_go

and_i_say_go_go_go:
   value: go go go
```

## Examples

Save these examples as `test.yml` to run them

### Echoing `hello` from a bash prompt

Test Plan:

```yaml
say_hello:
  bash: echo hello
```

Output:

```yaml
$ lorikeet test.yml
- name: say_hello
  pass: true
  output: |
    hello

  duration: 2.727446ms
```

### Matching the output of a bash command

Test Plan:

```yaml
say_hello:
  bash: echo hello
  matches: hello
```

Output:

```yaml
$ lorikeet test.yml
- name: say_hello
  pass: true
  duration: 2.68431ms
```

### Checking whether reddit is down

Test Plan:

```yaml
check_reddit:
  http: https://www.reddit.com
  matches: the front page of the internet
```

Output:

```yaml
$ lorikeet test.yml
- name: say_hello
  pass: true
  duration: 2.68431ms
```

### Logging into reddit

For configuration parameters of tests such as usernames and passwords, it makes sense to separate this out into a different file:

Config file:

```yaml
user: myuser
pass: mypass
```

Test Plan:

```yaml
login_to_reddit:
  http: 
    url: https://www.reddit.com/api/login/{{user}}
    form:
      user: {{user}}
      passwd: {{pass}}
      api_type: json
```

Output (Don't forget to specify the config file with `-c`) :

```yaml
$ lorikeet -c config.yml test.yml
- name: login_to_reddit
  pass: true
  output: {"json": {"errors": [], "data": {"need_https": true, "modhash": "....", "cookie": "..."}}}
  duration: 1420.8466ms
```



