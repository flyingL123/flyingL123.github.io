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

Here's a sample of the HTML that needs to be placed on the order confirmation page. As you can see, the red text is meant
to be replaced with the appropriate value for that specific order:

![Sample Google Trusted Stores Code]({{ site.base_url }}/assets/images/gts.png)
