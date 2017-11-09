---
title: "Drupal Commerce 2: Go to checkout after adding to cart"
description: How to redirect a customer after adding an item to their cart.
---

# Drupal Commerce 2: Go to checkout after adding to cart

There are times when you want to redirect a customer after some event.  For example, for some types of products you may want to redirect the customer directly to checkout instead of leave them on the current page (as is the default).

Two things need to happen:

1. Listen for the Cart module's `CART_ENTITY_ADD` event
2. Respond to the Symfony Kernel's `RESPONSE` event

## Step 1: Listen for the CartEvents::CART_ENTITY_ADD event

The [CartEntityAddEvent](https://github.com/drupalcommerce/commerce/blob/8.x-2.x/modules/cart/src/Event/CartEntityAddEvent.php) is [dispatched from the CartManager](https://github.com/drupalcommerce/commerce/blob/8.x-2.x/modules/cart/src/CartManager.php#L121) and includes the cart, purchased entity, quantity, and order item.

So we create a listener and look for the order item or purchased entity we are interested in.

{% highlight php %}
<?php

namespace Drupal\my_module\EventSubscriber;

use Drupal\commerce_cart\Event\CartEntityAddEvent;
use Drupal\commerce_cart\Event\CartEvents;
use Drupal\Core\Url;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpKernel\KernelEvents;
use Symfony\Component\HttpKernel\Event\FilterResponseEvent;

class CartEventSubscriber implements EventSubscriberInterface {

  /**
   * The request stack.
   *
   * @var \Symfony\Component\HttpFoundation\RequestStack
   */
  protected $requestStack;


  /**
   * CartEventSubscriber constructor.
   *
   * @param \Symfony\Component\HttpFoundation\RequestStack $request_stack
   *   The request stack.
   */
  public function __construct(RequestStack $request_stack) {
    $this->requestStack = $request_stack;
  }

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents() {
    $events = [
      CartEvents::CART_ENTITY_ADD => 'tryJumpToCheckout',
    ];
    return $events;
  }

  /**
   * Tries to jump to checkout, skipping cart after adding certain items.
   *
   * @param \Drupal\commerce_cart\Event\CartEntityAddEvent $event
   *   The add to cart event.
   */
  public function tryJumpToCheckout(CartEntityAddEvent $event) {
    $purchased_entity = $event->getEntity();
    if ($purchased_entity->bundle() === 'training_event') {
      $checkout_url = Url::fromRoute('commerce_checkout.form', [
        'commerce_order' => $event->getCart()->id(),
      ])->toString();
      $this->requestStack->getCurrentRequest()->attributes
        ->set('my_module_jump_to_checkout_url', $checkout_url);
    }
  }

}

{% endhighlight %}

Now we are listening for the cart event and adding our checkout url to the current request.  Next we need to listen for the `KernelEvents::RESPONSE` event where we can set a `RedirectResponse` using our url.

## Step 2: Respond to the KernelEvents::RESPONSE event

The [KernelEvents::RESPONSE](https://api.drupal.org/api/drupal/vendor%21symfony%21http-kernel%21KernelEvents.php/constant/KernelEvents%3A%3ARESPONSE/8.4.x) event is [dispatched by the Symfony HTTP kernel](https://api.drupal.org/api/drupal/vendor%21symfony%21http-kernel%21HttpKernel.php/function/HttpKernel%3A%3AfilterResponse/8.4.x) early in the page request life cycle and allows subscribers to alter the response that is sent back to the user.  In our case, we want the response to be a redirect.

So we add another listener:

{% highlight php %}
<?php

namespace Drupal\my_module\EventSubscriber;

use Drupal\commerce_cart\Event\CartEntityAddEvent;
use Drupal\commerce_cart\Event\CartEvents;
use Drupal\Core\Url;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpKernel\KernelEvents;
use Symfony\Component\HttpKernel\Event\FilterResponseEvent;

class CartEventSubscriber implements EventSubscriberInterface {

  /**
   * The request stack.
   *
   * @var \Symfony\Component\HttpFoundation\RequestStack
   */
  protected $requestStack;


  /**
   * CartEventSubscriber constructor.
   *
   * @param \Symfony\Component\HttpFoundation\RequestStack $request_stack
   *   The request stack.
   */
  public function __construct(RequestStack $request_stack) {
    $this->requestStack = $request_stack;
  }

  /**
   * {@inheritdoc}
   */
  public static function getSubscribedEvents() {
    $events = [
      CartEvents::CART_ENTITY_ADD => 'tryJumpToCheckout',
      KernelEvents::RESPONSE => ['checkRedirectIssued', -10],
    ];
    return $events;
  }

  /**
   * Tries to jump to checkout, skipping cart after adding certain items.
   *
   * @param \Drupal\commerce_cart\Event\CartEntityAddEvent $event
   *   The add to cart event.
   */
  public function tryJumpToCheckout(CartEntityAddEvent $event) {
    $purchased_entity = $event->getEntity();
    if ($purchased_entity->bundle() === 'training_event') {
      $checkout_url = Url::fromRoute('commerce_checkout.form', [
        'commerce_order' => $event->getCart()->id(),
      ])->toString();
      $this->requestStack->getCurrentRequest()->attributes
        ->set('my_module_jump_to_checkout_url', $checkout_url);
    }
  }

  /**
   * Checks if a redirect url has been set.
   *
   * Redirects to the provided url if there is one.
   *
   * @param \Symfony\Component\HttpKernel\Event\FilterResponseEvent $event
   *   The response event.
   */
  public function checkRedirectIssued(FilterResponseEvent $event) {
    $request = $event->getRequest();
    $redirect_url = $request->attributes->get('my_module_jump_to_checkout_url');
    if (isset($redirect_url)) {
      $event->setResponse(new RedirectResponse($redirect_url));
    }
  }

}

{% endhighlight %}

Finally, we need to register our listener with the container by adding it to our module's .services.yml file.

## Step 3: Add the event subscriber to the module's .services.yml file

The event subscriber needs to be be tagged as an `event_subscriber`, and it needs the `request_stack` service to be injected (see `core.services.yml`).  So we add the following to `my_module.services.yml`:

{% highlight yaml %}
services:
  ...
  
  my_module.cart_event_subscriber:
    class: Drupal\my_module\EventSubscriber\CartEventSubscriber
    arguments: ['@request_stack']
    tags:
      - { name: event_subscriber }

{% endhighlight %}

Now `drush cr` and you're good to go!
