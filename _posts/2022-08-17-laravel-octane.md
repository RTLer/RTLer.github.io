---
title: "Laravel+Octane"
date: 2022-08-14T00:00:00-04:00
categories:
  - blog
tags:
  - DevOps
---
For years Deploying PHP applications using nginx+fpm was the best practice and will be for some types of applications. But in modern environments like Kubernetes, it's not performing that well. Besides, it boots applications for every request (add something like 100ms to every request for applications using Laravel, for example)

## RoadRunner

RoadRunner is a PHP application server that runs your application in workers instead of running it for every request. And with a little bit of PSR7Worker magic, you can make your app ready to use RoadRunner.

The good news is that Laravel has an official package named Octane that You can install easily by requiring it:

```bash
composer require laravel/octane
php artisan octane:install --server=roadrunner
```

After installation, you can run it via:

```bash
php artisan octane:start --workers=2 --host=0.0.0.0 --port=8080 --max-requests=5000
```

Your application will be accessible at `http://0.0.0.0:8080`, easy as that.

```bash
  200    GET / ............... 120.36 ms
  200    GET / ............... 122.04 ms
  200    GET / ................. 3.08 ms
  200    GET / ................. 4.98 ms
  200    GET / ................. 2.96 ms
```

When refreshing the page, you can see that the first 2 requests are slower than the rest. That's because, during those requests, Roadrunner is booting the application in workers, and for the rest, it reuses those booted workers.

### RoadRunner plugins

RoadRunner has many [plugins](https://roadrunner.dev/docs/plugins-intro/2.x/en) you can leverage, for example, [health check](https://roadrunner.dev/docs/plugins-status/2.x/en). that can be used for running it in a containerized environment that we discussed in a future post.

## a personal experience

I used RoadRunner in a project that handles millions of requests per day, and with it and some production tweaking for it like `OPcache CLI` that you can find on its [official documentation](https://roadrunner.dev/docs/app-server-production/2.x/en) now that service uses about 20% of what it used to use before.

from `454 requests in 5s` to `2467 requests in 5s` (using go-wrk for HTTP benchmarking)

As for code just needed to ensure there is no static or global variable. You can read more about it in the managing [memory leaks section](https://laravel.com/docs/9.x/octane#managing-memory-leaks) of Laravel octane documents.

if you have any static variables or something that needs resetting, you can reset it before handling a new request by adding a listener to `octane.listeners.RequestReceived::class` like so, for example:

 ```php
       RequestReceived::class => [
            ...Octane::prepareApplicationForNextOperation(),
            ...Octane::prepareApplicationForNextRequest(),
            \App\Listeners\FlushStates::class, // <====
        ],
```

## links

- <https://roadrunner.dev/docs>
- <https://laravel.com/docs/9.x/octane>
