---
layout: post
title: Google Trusted Stores and Bigcommerce Without An Expensive Enterprise Account
disqus: true
---

[Bigcommerce](http://www.bigcommerce.com) has made it very difficult for merchants to integrate [Google Trusted Stores](https://www.google.com/trustedstores/).
An easy integration is only possible if you sign up for a very expensive Enterprise level account. These accounts can
cost much as $700/month; and that's the low end of the spectrum. For merchant's that qualify for Google Trusted Stores,
but don't have the budget or desire to pay so much money to Bigcommerce, this is extremely frustrating.

## The Problem

As part of the Google Trusted Stores program, merchant's need to install a code snippet on their checkout confirmation page.
This code snippet is called the [Order Confirmation Module](https://support.google.com/trustedstoresmerchant/answer/6063087).
The code is meant to send the details of the order to Google, which means the code needs to be customized for each invididual
order.

Here's a sample of the HTML that needs to be placed on the order confirmation page. As you can see, there are placeholders
between each `<span>` tag that are meant to be replaced with the appropriate value for that specific order:

~~~ html
<!-- START Google Trusted Stores Order -->
<div id="gts-order" style="display:none;" translate="no">

  <!-- start order and merchant information -->
  <span id="gts-o-id">MERCHANT_ORDER_ID</span>
  <span id="gts-o-email">CUSTOMER_EMAIL</span>
  <span id="gts-o-country">CUSTOMER_COUNTRY</span>
  <span id="gts-o-currency">CURRENCY</span>
  <span id="gts-o-total">ORDER_TOTAL</span>
  <span id="gts-o-discounts">ORDER_DISCOUNTS</span>
  <span id="gts-o-shipping-total">ORDER_SHIPPING</span>
  <span id="gts-o-tax-total">ORDER_TAX</span>
  <span id="gts-o-est-ship-date">ORDER_EST_SHIP_DATE</span>
  <span id="gts-o-est-delivery-date">ORDER_EST_DELIVERY_DATE</span>
  <span id="gts-o-has-preorder">HAS_BACKORDER_PREORDER</span>
  <span id="gts-o-has-digital">HAS_DIGITAL_GOODS</span>
  <!-- end order and merchant information -->

  <!-- start repeated item specific information -->
  <!-- item example: this area repeated for each item in the order -->
  <span class="gts-item">
    <span class="gts-i-name">ITEM_NAME</span>
    <span class="gts-i-price">ITEM_PRICE</span>
    <span class="gts-i-quantity">ITEM_QUANTITY</span>
    <span class="gts-i-prodsearch-id">ITEM_GOOGLE_SHOPPING_ID</span>
    <span class="gts-i-prodsearch-store-id">ITEM_GOOGLE_SHOPPING_ACCOUNT_ID</span>
  </span>
  <!-- end item 1 example -->
  <!-- end repeated item specific information -->

</div>
<!-- END Google Trusted Stores Order -->
~~~

In an ideal world, Bigcommerce would expose all of the order details through variables on the page. This would allow developers
to write some javascript code to loop through the order and construct the required HTML based on the order details. Unfortunately, Bigcommerce does not
expose most of these variables on the confirmation page unless you are an enterprise client. However, if you're determined,
there is a workaround.

## The Solution

The solution is to use the Bigcommerce API to fetch the information for the order, determine the value of each of the variables, and return the
appropriate HTML to the page. Here's how it works in our store:

The following code snippet is placed on the order confirmation page:

~~~ html
<script src="//my-server.com/google-trusted-stores-code/%%GLOBAL_OrderId%%"></script>
~~~

This snippet calls my server, passing it the Bigcommerce Order ID. Then, on my server, I fetch the order details
from the Bigcommerce API, and determine all of the information I need. For example, based on the products
in the order, I calculate the estimated ship date, and whether or not the order contains a preordered product.

Then, I pass all of this information to a view that construct the required HTML, and return it to the page
inside a call to `document.write()`.

~~~ php
<?php
  $html = \View::make('bigcommerce/googleTrustedCode', [
    'order'=>$order, 
    'shipDate'=>$shipDate,
    'hasPreorder'=>$hasPreorder
  ]);
  $html = preg_replace("/\r|\n/", "", $html);
  return Response::make('document.write("'.addslashes($html).'");');
?>
~~~

Here's what the view looks like:

~~~ html
<div id="gts-order" style="display:none;" translate="no">
    <span id="gts-o-id">{% raw %}{{ $order->id }}{% endraw %}</span>
    <span id="gts-o-domain">www.my-store-domain.com</span>
    <span id="gts-o-email">{% raw %}{{ $order->billing_email }}{% endraw %}</span>
    <span id="gts-o-country">{% raw %}{{ $order->billing_country_iso2 }}{% endraw %}</span>
    <span id="gts-o-currency">{% raw %}{{ $order->currency_code }}{% endraw %}</span>
    <span id="gts-o-total">{% raw %}{{ format_number($order->total_inc_tax) }}{% endraw %}</span>
    <span id="gts-o-discounts">0</span>
    <span id="gts-o-shipping-total">{% raw %}{{ format_number($order->shipping_cost_inc_tax) }}{% endraw %}</span>
    <span id="gts-o-tax-total">{% raw %}{{ format_number($order->total_tax) }}{% endraw %}</span>
    <span id="gts-o-est-ship-date">{% raw %}{{ $shipDate->addWeekday()->toDateString() }}{% endraw %}</span>
    <span id="gts-o-has-preorder">{% raw %}{{ $hasPreorder }}{% endraw %}</span>
    <span id="gts-o-has-digital">N</span>
    @foreach ($order->orderProducts as $op)
        <span class="gts-item">
            <span class="gts-i-name">{% raw %}{{ $op->product->name }}{% endraw %}</span>
            <span class="gts-i-price">{% raw %}{{ format_number($op->base_price) }}{% endraw %}</span>
            <span class="gts-i-quantity">{% raw %}{{ $op->quantity }}{% endraw %}</span>
            <span class="gts-i-prodsearch-id">{% raw %}{{ $op->product->ext_id }}{% endraw %}</span>
            <span class="gts-i-prodsearch-store-id">#######</span>
            <span class="gts-i-prodsearch-country">US</span>
            <span class="gts-i-prodsearch-language">en</span>
        </span>
    @endforeach
</div>
~~~

The code samples above are using PHP and Laravel, but the concepts apply to any language and framework. This strategy has been working
for our store for over a year now. We're able to avoid paying Bigcommerce for an enterprise account, but still successfully integrate Google Trusted Stores.

## Additional Notes

* This strategy will increase the page load time for your confirmation page. However, this is a tradeoff we are willing to live with.
A customer has reached the order confirmation page because they've already made a purchase. At this point in the customer experience
we are willing to accept a few extra milliseconds of page load time in order to integrate Google Trusted Stores.

* Bigcommerce has recently rolled out a new theme engine called [Stencil](https://stencil.bigcommerce.com/). I haven't looked into this
new system much yet, but it appears that it might be more flexible as far as what variables are available to developers on each page.
Hopefully this includes the order-specific variables needed to integrate Google Trusted Stores. If that is the case, this workaround will
no longer be necessary.


