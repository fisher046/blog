+++
draft = false
tags = ["Nginx","Kong"]
topics = ["OpenSource"]
description = "An example of how to use Kong"
title = "An example of Kong"
date = "2018-10-15T21:49:00+08:00"
+++

# What is Kong?

An API gateway based on Nginx like below picture. It allows programming in (1,2,3,4) location.

```
   +------------------------------+
   |                              |
   |          +-------+           |
   |    1     |       |      2    |
+------------->       +-------------->
   |          |       |           |
   |          | route |           |
   |    4     |       |      3    |
<-------------+       <--------------+
   |          | nginx |           |
   |          +-------+           |
   |            kong              |
   +------------------------------+

```

# An Example of Using Kong

## The Requirement

I need an API `/country`, it can return the deployment location which the service is running in.

For example,

{{< highlight shell >}}
$ curl http://localhost:30080/country
cannot know where it comes from

$ curl http://localhost:30080/country -H "location:US"
it comes from US
{{< /highlight >}}

It's meaningless because I need to add a header when calling the API. Let's see how to use Kong to resolve problems like this.

## Installation

Kong installation is very simple. But you need to install Cassandra or PosgreSQL. See more details [here](https://konghq.com/install/).

## Verify Installation

{{< highlight shell >}}
$ curl http://localhost:8001
{{< /highlight >}}

## Add a Service

Service - Kong uses it to refer to the upstream APIs and microservices it manages.

{{< highlight shell >}}
$ curl http://localhost:8001/services -d "name=sample" -d "url=http://localhost:30080"
{{< /highlight >}}

## Add a Route for the Service

Route - It specifies how (and if) requests are sent to their Services after they reach Kong. A single Service can have many Routes.

{{< highlight shell >}}
$ curl http://localhost:8001/services/sample/routes -d "hosts[]=service1.com"
{{< /highlight >}}

Now we can call the API like:

{{< highlight shell >}}
$ curl http://localhost:8000/country -H "host:service1.com" -H "location:US"
it comes from US
{{< /highlight >}}

## Add a Plugin to the Route

Plugin - It allows you to easily add new features to your API or make your API easier to manage.

Now we add an existing plugin named "request transformer". It can add header to API call based on domain name.

{{< highlight shell >}}
$ curl http://localhost:8001/routes/735b47c7-af4d-455f-846f-f78a71408fbb/plugins -d "name=request-transformer" -d "config.add.headers=location:US"
{{< /highlight >}}

Now we can call the API like:

{{< highlight shell >}}
$ curl http://localhost:8000/country -H "host:service1.com"
it comes from US
{{< /highlight >}}

Then add the second route and add a similar plugin to the route.

{{< highlight shell >}}
$ curl http://localhost:8001/services/sample/routes -d "hosts[]=service2.com"

$ curl http://localhost:8001/routes/3ef1cf68-c513-4937-8b88-8a1d5d9d54ab/plugins -d "name=request-transformer" -d "config.add.headers=location:UK"
{{< /highlight >}}

And add below line to `/etc/hosts`.

```
127.0.0.1    service1.com service2.com
```

Then we reach the target now.

{{< highlight shell >}}
$ curl http://service1.com:8000/country
it comes from US

$ curl http://service2.com:8000/country
it comes from UK
{{< /highlight >}}

# Benefits

- So many community provided plugins.
- Can write your own plugins.
- Dynamically add/remove service proxy without restarting.
