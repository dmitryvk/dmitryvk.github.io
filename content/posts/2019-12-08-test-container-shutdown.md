---
title: "Notes on guaranteed shutdown of test containers"
slug: test-container-shutdown
description: null
date: 2019-12-08T03:15:07+03:00
lastmod: 2019-12-19T22:22:00+03:00
type: posts
draft: false
categories:
- General
tags:
-
series:
-
---

One use for containers is to provision dependencies for automated software testing.

This approach is supported by at least several libraries for several languages. These libraries are usually easy to set up and easy to use.

Usually, one would start the container in `setUp` and stop the container in `tearDown`. This approach usually works fine but it does one quite big drawback: **if a test crashes then you leak the container**.

In my prior experiences with provisioning dependencies for automated tests, it is very easy to exhaust resources when debugging tests or when tests fail to clean up after themselves. I believe that tests should be as robust as possible so I find this situation unacceptable.

I had not seen any mention of this in documentation so I decided to share what I use to prevent this issue.

I had tried different techniques to overcome this issue and I had settled on the following technique:

1.  All containers are started with `autoremove` option to ensure that storage is cleaned up when the container is stopped.
    But this is not enough - we need to ensure that the container will actually be stopped.
2.  When the container starts, the watchdog is injected which monitors whether there is an active user of the container.
3.  If watchdog detects that test software has unexpectedly disconnected from container then it forcibly terminates the container.
4.  One of the easy and robust ways of implementing watchdog is the following:
    1.  A shell script is injected into a running container that reads from standard input until it is closed.

        ```bash
        # this loop will keep running while tests are attached to the container
        # when docker detects client disconnect the loop will be terminated
        while read VAR; do
          true;
        done;

        kill -9 1 # or some other command that will terminate the container
        ```
    2.  Background task inside the test process starts and attaches to the injected script.

This reliably ensures that tests don't leave unused containers behind.

Some container management libraries do not even try solve this issue (e.g., popular .NET library [TestContainers-dotnet](https://github.com/testcontainers/testcontainers-dotnet)).

[TestContainers](https://testcontainers.org) (a java library) uses different approach for ensuring guaranteed shutdown and release of all used resources. It starts a separate container (called `ryuk`) and establishes persistent connection to it; `ryuk` then uses this connection to monitor presence of its client. I haven't used it extensively, but apparently it works really well. Note that this is not mentioned in `TestContainers`' documentation and I even missed this very important feature on my first review of such libraries.

(Updated on 2019-12-19 to mention `ryuk`)
