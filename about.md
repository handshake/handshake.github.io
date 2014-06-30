---
layout: page
title: Who We Are
permalink: /about/
image: /img/eng.png
---

Handshake Engineering is a team of technologists dedicated to connecting great brands to local retailers with great software. The software we build replaces the antiquated paper-and-clipboard-based systems used by brands and their salespeople, allowing for more products to reach more retailers with less cost, at greater speed and with less environmental impact.

### Our Challenges

Increasing the reach of great brands, and improving the lives of their team members is a big challenge. Solving this one problem requires great UX, extremely fast and extensible data-stores, and a framework allowing for trivial integration with their existing tools.

#### Faster Than The Order Form

Before Handshake, the state-of-the-art in wholesale sales was an order form on a clipboard. Orders were taken by hand, faxed, and hand-entered into an [ERP](http://en.wikipedia.org/wiki/Enterprise_resource_planning) by data-entry specialists. Providing a better experience starts with being simpler and faster than pen and paper. Sounds simple right? Well, it turns out that taking an order with pen on an order form is pretty fast, especially when you use short-hand, and even though paper is massively expensive and inefficient to process, you have to beat the upfront speed to win users. This one challenge leads us to continuously explore new and increasingly efficient interfaces for our users.

#### Integration

Our software lives in an ecosystem of software used by brands to manage all aspects of their businesses. Building robust and well-documented APIs is not just a nice-to-have, it's mission critical. Along with powering custom integrations, our APIs support out-of-the-box integrations with platforms like Salesfoce and QuickBooks, allowing our products to painlessly and seamlessly work with our customers' existing infrastructure.

#### Micro-Services #FTW

Providing robust APIs, with the flexibility to experiment with increasingly nuanced UX requires a solid foundation. Starting from a monolithic Django application, we will be breaking down our back-end into increasingly smaller REST services, each backed by the right language and framework for the job.

### Our Stack

We try to use the right tool for the job, and as few tools as are necessary to get the job done. Our minimalistic stack is optimized for heterogony, flexibility and interoperability. We reserve the right to change our mind -- and our stack -- to meet the needs of our emerging problems:

 * Our inf is built on `managed hosting`, but will soon be on `AWS` or `RackSpace`
 * We use `salt` for configuration management.
 * We use `fab` to deploy.
 * We use `git` for source control.
 * Our database is `PostgreSQL`, and we also use `Redis`.
 * We have one big `Django` app, but are moving towards REST micro-services.
 * We are nearly done porting to a single universal `iOS` app.
 * We use `Backbone.js` on the Front-End.
 * We also use `CoffeeScript`, though there is some debate about this.
