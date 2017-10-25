---
title: Developing and Contributing to Drupal Commerce 2
description: An article about Developing and Contributing to Drupal Commerce 2
---

# Developing and Contributing to Drupal Commerce 2

I've recently been working on migrating a legacy e-commerce system to a new Drupal 8 / Commerce 2 platform.  In the process of deploying four new stores featuring product bundles, stock management, custom pricing and event registrations, I've contributed to dozens of core and contributed issues leading to the release of Commerce 2.0 alongside a group of talented and dedicated developers around the world.

I'm so impressed with the level of attention, care and love put into Commerce 2.x and am so thrilled to be working on such a flexible and powerful platform with such an active community of dedicated developers! I'd like to highlight the things I've learned while working on my client stores and the Commerce project, and encourage businesses interested in Drupal to consider Drupal Commerce as their next e-commerce platform.

## About the project, or "Hacking away at technical debt and making room to grow"

### Problem/motivation

The company had up until this point succeeded for several years on a business model focused primarily on direct response (TV advertising) phone-based sales and batch order processing with offsite fulfillment.  But the business was becoming increasingly focused on online sales of digital content and event registrations in addition to a growing number of physical product lines.

The legacy system originated from a thin web layer including various hand-made shopping carts that were tailored for each order type and product line, all tied in various ways to offline and manual processes for order tracking, product management, customer support and business intelligence.

Many of the issues with the legacy system related in one way or another to the fact that all orders, for both physical and digital products, were processed by a third-party fulfillment center that had remained the center of their e-commerce operations even as their focus shifted away from physical goods and more toward digital products and event registrations.  The fees, technical limitations, and data inefficiencies became painfully apparent as their business grew in products, scale, and complexity.

The company wanted to increase their throughput of event and product sales and at the same time decrease their human workload, data problems, expenses and customer support issues.  They needed greater visibility into the state of a growing number of interconnected business operations including sales, product and promotion management, customer relationship and external integration. The most important issues to resolve included:

1. **Data efficiency** — having the data needed to make informed business decisions.
2. **Workflow reliability** — having processes in place to reduce human workload and errors.
3. **Growth capability** — having the architecture for flexibility in sales, services and support.

## Proposal

Since the company had already standardized on Drupal as their publishing platform and members-site system of choice for most of their projects, and since the team already understood that Drupal provides unequaled user management and data modeling, Drupal Commerce (7.x and 8.x) was considered alongside Magento, Woo Commerce and other open source platforms as well as any other proprietary or licensed e-commerce systems that might have fit the bill.

Having previously worked with the company on digital access and members' site projects, I understood the growing needs for flexibility and the various aspects of the existing systems which would be easily integrated and optimized with Drupal and Commerce—especially 2.x, which at that time was only at beta1.

I proposed Drupal Commerce 2.x, and specifically to use Commerce 2.x over 1.x, for the following primary reasons:

1. **Multi-store** — the company had already dealt with many issues supporting multiple stores, and the [Store architecture](https://docs.drupalcommerce.org/commerce2/developer-guide/stores) in Commerce 2.x suited their needs perfectly.
2. **Order types and Checkout flows** — the company relied on customized [order types](https://docs.drupalcommerce.org/commerce2/developer-guide/orders/order-types) and tailored [checkout](https://docs.drupalcommerce.org/commerce2/user-guide/checkout) flows, and these would only grow more complex.
3. **Entities, Events, and Plugins** — (The basis of 1. and 2.) I realized the potential of entities everywhere, with their display and form modes, of event systems, plugins and pluggable services, plus other Drupal 8/Symfony 2 features that would make the growing needs for customization easy to scale in a professional way.
4. **Our timeline and the Commerce development cycle** — I knew that committing to the bleeding edge of Commerce early in the development cycle would help motivate the development of 2.x with use cases and contributions from our project, and would help my understanding of Commerce 2.x as the core project grew.

### Completed Tasks, Or "What I had to figure out along the way"

**Payment processing** - One of the requirements was that we use Vantiv for payment processing, and specifically that we use their _eProtect_ token implementation.  While D7 had a working Commerce Vantiv module, it did not implement eProtect and was not very well suited to be "upgraded" to eProtect. I knew I'd be rewriting the D7 version if I had to use it.

At alpha1, Commerce 2 already had the Onsite Payments API complete, so while the rest of Commerce's APIs were still under development I started rewriting [Commerce Vantiv](https://www.drupal.org/project/commerce_vantiv) for 8.x, which became one of the first [payment gateways for Commerce 2.x](https://docs.drupalcommerce.org/commerce2/developer-guide/payments/gateways-providers), and is now at a stable version 2.1.

**Stock management** - I was happy to find that Guy Schneerson had already started 8.x work on [Commerce Stock](http://cgit.drupalcode.org/commerce_stock/tree/?h=8.x-1.x).  The module still needs schema and API stabilization, but the first thing I did was [clean up the code and setup automated testing](https://github.com/BBGuy/commerce_stock/pull/1) so we could move forward with more confidence in our changes.

Over the course of this client project, I contributed to the early development of Commerce Stock with support all [purchasable entities](https://docs.drupalcommerce.org/commerce2/developer-guide/products/purchasable-entities) (for product bundles), to create stock transactions on order events, and other issues toward a [beta1 release](https://www.drupal.org/node/2834966).  We still need to [add support for multiple stores and clean up the locations-store relationship](https://www.drupal.org/node/2858391), and in Commerce [finalize the availability manager service](https://www.drupal.org/node/2710107).

**Product bundles** - Like many stores, we needed the ability to bundle several items into one single purchasable item with their own add to cart forms, displays, skus, and prices.

So I started work with Olaf Karsten on the 8.x-1.x branch of [Commerce Product Bundle](https://www.drupal.org/node/2799643) after researching different use cases, distinguishing between essentially 'static' vs. 'dynamic' bundles, and writing the first proof of concept for the static bundle use case.  And we needed to [integrate product bundles with Commerce Stock](https://www.drupal.org/node/2846501), so I created a submodule that proxies availability checks to all items in a bundle.

While the project is still very much -dev, within a few months we've built the basic functionality for creating, administering, and purchasing static product bundles, and feel that the basic entity architecture should well support the upcoming work on support for dynamic or user-chosen bundles.  See ongoing [discussion about the 8.x version](https://www.drupal.org/node/2799643) and [all open 8.x-1.x issues for Commerce Product Bundle](https://www.drupal.org/project/issues/commerce_product_bundle?text=&status=Open&version=8.x).

**Special pricing** - We needed to provide pricing based on several different criteria.  By implementing a few custom 'commerce_price.price_resolver' services, we were able to accomplish with a just few fields and php classes, instead of something like a Commerce Price List module.

**Promotions** - The new [Promotions and Coupons](https://docs.drupalcommerce.org/commerce2/user-guide/promotions) in Commerce 2 made it easy to consolidate our old offer codes system (don't ask!) into simple coupons in combination with special pricing.

**Custom reporting** - There was no [Commerce Reports](https://www.drupal.org/project/commerce_reports) for 8.x at the time, so I created a few custom controllers to generate reports with CSV exports for Sales, Tax, and Inventory filtered by Store and date range. Using Composer made it easy for me to create and use [SBA Geodata](https://www.drupal.org/sandbox/steveoliver/2852343), a simple Drupal service module that wraps calls to US Small Business Association (SBA) web services for county data used in some of the tax reports.

**Event sales and scheduling workflows** - I created administrative workflows for drafting, approving, and publishing events for registration and purchase, and created customer workflows for registering for events, with the ability for managers to track attendance and promotions for the event.

The events for purchase are types of products, with their own order item types and checkout flows, which are amazing. But since [products don't support revisions](https://www.drupal.org/node/2656896), I could not take advantage of [Drupal's Workflow initiative](https://www.drupal.org/node/2721129) to implement our editorial workflow for these event products.  For now, boolean fields and [state_machine](https://www.drupal.org/project/state_machine) achieve roughly the same thing.

One of the most exciting aspects of this project has been this automation and optimization of the variations that occur in the workflow of event scheduling and registrations, since it is responsible for generating the greatest increase in operational efficiency and effectiveness for the business and its customers.

### Looking forward to the next Commerce 2 release

Through the development of this client project--now at it's second second major milestone--I have built a great amount of functionality on top of the careful and hard work of Commerce Guys' [Bojan](https://www.drupal.org/u/bojanz) and [Matt](https://www.drupal.org/u/mglaman), with the help of a few other key contributors like [Olaf Karsten](https://www.drupal.org/u/olafkarsten) and [Guy Schneerson](https://www.drupal.org/u/guy_schneerson), working directly with the new standards, testing approaches, and APIs of Drupal 8 and Commerce 2.x.

The whole time, I was pleasantly surprised by the elegance of the Commerce 2.x architecture and was always confident in the implicit development practices of Drupal 8.  I am grateful to have been involved in what seems like the perfect Drupal 8 development project, to have found help and inspiration from big issues to small interactions on Slack, IRC and in issue queues.

I'm excited to see the new flood of interest in Commerce 2.0 and look forward to [the next release](https://www.drupal.org/node/2913801)!
