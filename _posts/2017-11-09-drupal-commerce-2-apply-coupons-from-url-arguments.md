---
title: "Drupal Commerce 2: Apply coupons from URL arguments."
description: How to apply coupons to an order from URL arguments in Drupal Commerce 2.
---

# Drupal Commerce 2: Apply Coupons from URL Arguments

If you decide not to enable the Coupon Redemption pane in your checkout flow, but still want to allow some customers to redeem their coupons somehow, you can allow them to be passed via a URL argument.

Two things need to happen:

1. Look for coupon codes in a specific URL argument (`?coupons=ABC123,DEF456`) and save them to the users's session.
2. Implement an order processor that can check for those codes and apply them to an order (if and when an order/cart exists for the session).

## Step 1: Listen for the KernelEvents::REQUEST event

The [KernelEvents::REQUEST](https://api.drupal.org/api/drupal/vendor!symfony!http-kernel!KernelEvents.php/constant/KernelEvents%3A%3AREQUEST/8.4.x) event is [dispatched by the Symfony HTTP kernel](https://api.drupal.org/api/drupal/vendor%21symfony%21http-kernel%21HttpKernel.php/function/HttpKernel%3A%3AhandleRaw/8.4.x) in the first part of the page request life cycle and allows subscribers to alter the request that is sent to Drupal.  In our case, we want to check if the request contains a `coupons` parameter.

So we'll create and register one basic event subscriber that looks for the query arg and saves it for later.

We'll call it 'my_module.url_coupon_subscriber'.

`my_module.services.yml`:
{% highlight yaml %}
services:
  ...
  my_module.url_coupon_subscriber:
    class: Drupal\my_module\UrlCouponSubscriber
    arguments: ['@request_stack', '@entity_type.manager', '@session']
    tags:
      - { name: event_subscriber }
{% endhighlight %}

`UrlCouponSubscriber.php`:
{% highlight php %}
<?php

namespace Drupal\my_module;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\Session\SessionInterface;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpKernel\Event\GetResponseEvent;
use Symfony\Component\HttpKernel\HttpKernelInterface;
use Symfony\Component\HttpKernel\KernelEvents;

class UrlCouponSubscriber implements EventSubscriberInterface {

  /**
   * The current request.
   *
   * @var \Symfony\Component\HttpFoundation\Request
   */
  protected $request;

  /**
   * The session.
   *
   * @var \Symfony\Component\HttpFoundation\Session\SessionInterface
   */
  protected $session;

  /**
   * The coupon storage.
   *
   * @var \Drupal\commerce_promotion\CouponStorageInterface
   */
  protected $couponStorage;

  /**
   * Constructs a new UrlCouponSubscriber.
   *
   * @param \Symfony\Component\HttpFoundation\RequestStack $request
   *   The request stack.
   * @param \Drupal\Core\Entity\EntityTypeManagerInterface $entity_type_manager
   *   The entity type manager.
   * @param \Symfony\Component\HttpFoundation\Session\SessionInterface $session
   *   The session.
   *
   * @throws \Exception
   */
  public function __construct(RequestStack $request, EntityTypeManagerInterface $entity_type_manager, SessionInterface $session) {
    $this->request = $request->getCurrentRequest();
    $this->couponStorage = $entity_type_manager->getStorage('commerce_promotion_coupon');
    $this->session = $session;
  }

  /**
   * Sets coupon codes in the user's session if the query param is present.
   *
   * @param \Symfony\Component\HttpKernel\Event\GetResponseEvent $event
   *   The Event to process.
   */
  public function onKernelRequestSetUrlCoupons(GetResponseEvent $event) {
    if ($event->getRequestType() == HttpKernelInterface::MASTER_REQUEST) {
      $query = $event->getRequest()->query;
      if ($query->has('coupons')) {
        // Expect a string of comma-separated coupon codes.
        // For sanity, filter out anything but alphanumeric and ','.
        $coupons_arg = $query->get('coupons');
        $clean_arg = preg_replace('/[^[:alnum:],]/', '', $coupons_arg);
        $coupon_codes = explode(',', $clean_arg);
        $this->session->set('my_module_url_coupon_codes', $coupon_codes);
      }
    }
  }

  /**
   * Gets the coupon codes from the session.
   *
   * @param bool $clear
   *   TRUE (default) if the codes should be cleared.
   *
   * @return \Drupal\commerce_promotion\Entity\CouponInterface[]
   *   All coupons for the session.
   */
  public function getUrlCoupons($clear = TRUE) {
    $coupons = [];
    if ($this->session->has('my_module_url_coupon_codes')) {
      foreach ($this->session->get('my_module_url_coupon_codes') as $code) {
        if ($coupon = $this->couponStorage->loadEnabledByCode($code)) {
          $coupons[] = $coupon;
        }
      }
      if ($clear) {
        $this->session->remove('my_module_url_coupon_codes');
      }
    }

    return $coupons;
  }

  /**
   * Registers the methods in this class that should be listeners.
   *
   * @return array
   *   An array of event listener definitions.
   */
  public static function getSubscribedEvents() {
    $events[KernelEvents::REQUEST][] = ['onKernelRequestSetUrlCoupons'];

    return $events;
  }

}

{% endhighlight %}

If the parameter exists, we just want to save the coupon codes in the current user's session.  We don't neccessarily want to load them when they are passed in.  We may not ever have an order, so all we want to do is save the coupon codes for now.  But knowing that later we will want to check if coupons exist, are valid, and to be able to apply them to an order, there's a `::getUrlCoupons` method.

## Step 2: Implement an order processor to apply the coupons.

By tagging a service 'commerce_order.order_processor', the Order module will allow our code to respond to the [Order Refresh](https://github.com/drupalcommerce/commerce/blob/8.x-2.x/modules/order/src/OrderRefresh.php#L155) process and make changes to the order.

In this order processor, we'll want to check for those coupon codes we saved in the event subscriber.  So we inject our 'my_module.url_coupon_subscriber' service dependency in the `arguments: []` section of the order processor's service.

Also, it is important to notice that the `priority` is set to 60.  This is to make sure this processor runs before the [PromotionOrderProcessor](https://github.com/drupalcommerce/commerce/blob/8.x-2.x/modules/promotion/commerce_promotion.services.yml#L10) that actually checks for related Promotions and applies them to the order.

`my_module.services.yml`:
{% highlight yaml %}
services:
  ...
  my_module.order_process.add_url_coupons:
      class: Drupal\my_module\OrderProcessor\AddUrlCoupons
      arguments: ['@my_module.url_coupon_subscriber']
      tags:
        - { name: commerce_order.order_processor, priority: 60 }
{% endhighlight %}

`AddUrlCoupons.php`:
{% highlight php %}
<?php

namespace Drupal\my_module\OrderProcessor;

use Drupal\commerce_order\Entity\OrderInterface;
use Drupal\commerce_order\OrderProcessorInterface;
use Drupal\my_module\UrlCouponManager;

/**
 * Applies any coupons present in the current session.
 *
 * @package my_module
 */
class AddUrlCoupons implements OrderProcessorInterface {

  /**
   * The url coupon manager.
   *
   * @var \Drupal\my_module\UrlCouponSubscriber
   */
  protected $urlCouponSubscriber;

  /**
   * SplitEventItems constructor.
   *
   * @param \Drupal\my_module\UrlCouponManager $url_coupon_manager
   */
  public function __construct(UrlCouponManager $url_coupon_manager) {
    $this->urlCouponSubscriber = $url_coupon_manager;
  }

  /**
   * {@inheritdoc}
   */
  public function process(OrderInterface $order) {
    if ($coupons = $this->urlCouponSubscriber->getUrlCoupons()) {
      foreach ($coupons as $coupon) {
        $order->get('coupons')->appendItem($coupon);
      }
    }
  }

}

{% endhighlight %}

Drush CR. Done.
