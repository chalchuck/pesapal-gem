---
title: Get
layout: default
---


Synopsis
--------

Basically it's a gem that makes it easy to integrate your app with
[Pesapal's][24] payment gateway. It Handles all the [oAuth stuff][1] abstracting
any direct interaction with the API endpoints so that you can focus on what
matters. _Building awesome_.

The gem should be [up on RubyGems.org][7], it's [accompanying RubyDoc reference
here][13], the [CHANGELOG here][21] and [all the releases here][12].

_Ps: No 3rd party oAuth library dependencies, it handles all the oAuth flows on
it's own so your app is one dependency less._


Installation
------------

Add this line to your application's Gemfile:

    gem 'pesapal'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install pesapal

For Rails, you need to run the generator to create sample pesapal.yml file:

    rails generate pesapal:install


Usage
-----


### Initialization ###

There are 2 ways to initialize the Pesapal object:

1. YAML config at `/config/pesapal.yml`
2. Config hash

Initialize Pesapal object and choose the environment, there are two environments;
`:development` and `:production`. They determine if the code will interact
with the testing or the live Pesapal API.

```ruby
# Sets environment intelligently to 'Rails.env' (if Rails) or :development (if non-Rails)
pesapal = Pesapal::Merchant.new

# Sets environment to :development
pesapal = Pesapal::Merchant.new(:development)

# Sets environment to :production
pesapal = Pesapal::Merchant.new(:production)
```

####Option 1####

In the above case, the configuration has already been loaded (at application
start by initializer) from a YAML file located at `"/config/pesapal.yml"`. The
appropriate credentials are picked depending on set environment.

This is the recommended method if using Rails.


####Option 2####

If you do not wish to use the YAML config method then the object is set up with
some bogus credentials which would not work anyway and therefore, the other
option is that you set them yourself. Which, you can do using a hash as shown
below (please note that Pesapal provides different keys for different
environments and since this is like an override, there's the assumption that you
chose the right one).

Recommended if not using Rails.

```ruby
# set pesapal api configuration manually (override YAML & bogus credentials)
pesapal.config = {  :callback_url => 'http://0.0.0.0:3000/pesapal/callback',
                    :consumer_key => '<YOUR_CONSUMER_KEY>',
                    :consumer_secret => '<YOUR_CONSUMER_SECRET>'
                  }
```

_Ps: You can change the environment using `pesapal.set_env(:development)`
(example) if for some reason you want to override what was set in the
constructor. This method also changes the API endpoints appropriately._


###YAML Configuration###

The YAML file should look something like this. If you ran the generator you
should have it already in place with some default values. Feel free to change
them appropriately.

```yaml
development:
  callback_url: 'http://0.0.0.0:3000/pesapal/callback'
  consumer_key: '<YOUR_DEV_CONSUMER_KEY>'
  consumer_secret: '<YOUR_DEV_CONSUMER_SECRET>'

production:
  callback_url: 'http://1.2.3.4:3000/pesapal/callback'
  consumer_key: '<YOUR_PROD_CONSUMER_KEY>'
  consumer_secret: '<YOUR_PROD_CONSUMER_SECRET>'
```


### Posting An Order ###

Once you've finalized the configuration, set up the order details in a hash as
shown in the example below ... all keys **MUST** be present. If there's one that
you wish to ignore just leave it with a blank string but make sure it's included
e.g. the phonenumber.

```ruby
#set order details
pesapal.order_details = { :amount => 1000,
                          :description => 'this is the transaction description',
                          :type => 'MERCHANT',
                          :reference => '808-707-606',
                          :first_name => 'Swaleh',
                          :last_name => 'Mdoe',
                          :email => 'user@example.com',
                          :phonenumber => '+254722222222',
                          :currency => 'KES'
                        }
```

Then generate the transaction url as below. In the example, the value is
assigned to the variable `order_url` which you can pass on to the templating
system of your choice to generate an iframe. Please note that this method
utilizes all that information set in the previous steps in generating the url so
it's important that it's the last step in the post order process.

```ruby
# generate transaction url
order_url = pesapal.generate_order_url

# order_url will a string with the url example;
# http://demo.pesapal.com/API/PostPesapalDirectOrderV4?oauth_callback=http%3A%2F%2F1.2.3.4%3A3000%2Fpesapal%2Fcallback&oauth_consumer_key=A9MXocJiHK1P4w0M%2F%2FYzxgIVMX557Jt4&oauth_nonce=13804335543pDXs4q3djsy&oauth_signature=BMmLR0AVInfoBI9D4C38YDA9eSM%3D&oauth_signature_method=HMAC-SHA1&oauth_timestamp=1380433554&oauth_version=1.0&pesapal_request_data=%26lt%3B%3Fxml%20version%3D%26quot%3B1.0%26quot%3B%20encoding%3D%26quot%3Butf-8%26quot%3B%3F%26gt%3B%26lt%3BPesapalDirectOrderInfo%20xmlns%3Axsi%3D%26quot%3Bhttp%3A%2F%2Fwww.w3.org%2F2001%2FXMLSchema-instance%26quot%3B%20xmlns%3Axsd%3D%26quot%3Bhttp%3A%2F%2Fwww.w3.org%2F2001%2FXMLSchema%26quot%3B%20Amount%3D%26quot%3B1000%26quot%3B%20Description%3D%26quot%3Bthis%20is%20the%20transaction%20description%26quot%3B%20Type%3D%26quot%3BMERCHANT%26quot%3B%20Reference%3D%26quot%3B808%26quot%3B%20FirstName%3D%26quot%3BSwaleh%26quot%3B%20LastName%3D%26quot%3BMdoe%26quot%3B%20Email%3D%26quot%3Bj%40kingori.co%26quot%3B%20PhoneNumber%3D%26quot%3B%2B254722222222%26quot%3B%20xmlns%3D%26quot%3Bhttp%3A%2F%2Fwww.pesapal.com%26quot%3B%20%2F%26gt%3B
```

_Ps: Please note the `:callback_url` value in the `pesapal.config` hash ...
after the user successfully posts the order, the response will be sent to this
url. Refer to [official Pesapal Step-By-Step integration guide][18] for more
details._


### Querying Payment Status ###

Use this to query the status of the transaction. When a transaction is posted to
Pesapal, it may be in a PENDING, COMPLETED, FAILED or INVALID state. If the
transaction is PENDING, the payment may complete or fail at a later stage.

Both the unique merchant reference generated by your system (compulsory) and the
pesapal transaction tracking id (optional) are input parameters to this method
but if you don't ensure that the merchant reference is unique for each order on
your system, you may get INVALID as the response. Because of this, it is
recommended that you provide both the merchant reference and transaction
tracking id as parameters to guarantee uniqueness.

```ruby
# option 1: using merchant reference only
payment_status = pesapal.query_payment_status("<MERCHANT_REFERENCE>")

# option 2: using merchant reference and transaction id (recommended)
payment_status = pesapal.query_payment_status("<MERCHANT_REFERENCE>","<TRANSACTION_ID>")
```


### Querying Payment Details ###

Same as querying payment status above, but the return value contains more
information (and is a hash as opposed to a string).

```ruby
# pass in merchant reference and transaction id
payment_details = pesapal.query_payment_details("<MERCHANT_REFERENCE>","<TRANSACTION_ID>")
```

The result is a hash that looks something like this ...

```
{
  :method => "<PAYMENT_METHOD>",
  :status => "<PAYMENT_STATUS>",
  :merchant_reference => "<MERCHANT_REFERENCE>",
  :transaction_tracking_id => "<TRANSACTION_ID>"
}
```


### IPN Listening ###

Use the `ipn_listener` method to listen to Pesapal IPN calls to easily create an
appropriate response, example below.

```ruby
# pass in the notification type, merchant reference and transaction id
response_to_ipn = pesapal.ipn_listener("<NOTIFICATION_TYPE>", "<MERCHANT_REFERENCE>","<TRANSACTION_ID>")
```

The variable, `response_to_ipn`, now holds a response as the one shown below.
Using the status you can customise any actions (e.g. database inserts and
updates) and finally, it's up to you to send the `:response` back to pesapal. The
hard part is done for you.

```
{
  :status => "<PAYMENT_STATUS>",
  :response => "<IPN_RESPONSE>"
}
```

_Ps: Refer to Pesapal official documentation to make sure you understand what
data Pesapal sends to IPN and what result they expect back._


Support & Issues
----------------

In case you are having any issues using this gem please **_do not_** email me
directly, I'd rather you [submit new issues (and even requests) here][23] ...
obviously after [checking if the issue has already been raised and closed][6].
This way, other people get to share in the conversation that would have been
private and out of their reach.


Contributing
------------

### To gem's code

1. Make sure you've read the [M.O. ★][14] ([blog article here][16]) especially
   [the part about my conventions when writing and merging new features][15].
2. [Fork it][8]
3. Create your feature branch (`git checkout -b BRANCH_NAME`).
4. Make your changes, write tests for them if necessary & run `bundle exec rspec spec`. Tests ideally should pass.
5. Commit your changes (`git commit -am 'AWESOME COMMIT MESSAGE'`).
6. Push to the branch (`git push origin BRANCH_NAME`).
7. Create new pull request and we can [have the conversations here][17].


### To gem's documentation

Gem documentation is of two types, feel free to contribute to any;

1. Gem home page at [itsmrwave.github.io/pesapal-gem][26] _(simple step-by-step)_
2. API documetation generated by [Yard][25] _(very comprehensive)_

Gem home page is built on [Jekyll][27], which is a fun and easy to use static
site generator and [hosted on GitHub Pages][30]. The code is in the [`gh-branch`][29]
if you want to have a peek.

Preview the gem API documentation locally by installing [Yard][25] (`gem install
yard`) and call `$ yard server` to set up a local documentation server usually
running on `http://0.0.0.0:8808`. The result should be similar to the API
documention up on [rubydoc.info/gems/pesapal][13].

_Ps: If you've written code and can't figure out why they aren't passing (or if
you've written your own tests ... just send a pull a request. We can then do a
[code review][32])_.


Testing
-------

Tests run [RSpec][31] which is a Behaviour-Driven Development tool. To run
tests, call;

```bash
$ bundle exec rspec spec
```


References
----------

* [oAuth 1.0 Spec][1]
* [Developing a RubyGem using Bundler][2]
* [RailsGuides][20]
* [Make your own gem][3]
* [Getting started with RSpec][33]
* [Pesapal API Reference (Official)][4]
* [Pesapal Step-By-Step Reference (Official)][18]
* [Pesapal PHP API Reference (Unofficial)][5]


License
-------

[King'ori J. Maina][10] © 2014. The [MIT License bundled therein][11] is a
permissive license that is short and to the point. It lets people do anything
they want as long as they provide attribution and waive liability.


[1]: http://oauth.net/core/1.0/
[2]: https://github.com/radar/guides/blob/master/gem-development.md
[3]: http://guides.rubygems.org/make-your-own-gem/
[4]: http://developer.pesapal.com/how-to-integrate/api-reference
[5]: https://github.com/itsmrwave/pesapal-php#pesapal-php-api-reference-unofficial
[6]: https://github.com/itsmrwave/pesapal-gem/issues?state=closed
[7]: https://rubygems.org/gems/pesapal
[8]: https://github.com/itsmrwave/pesapal-gem/fork
[9]: https://github.com/itsmrwave/pesapal-gem#contributing--testing
[10]: http://kingori.co/
[11]: https://raw.githubusercontent.com/itsmrwave/pesapal-gem/master/LICENSE.txt
[12]: https://github.com/itsmrwave/pesapal-gem/releases/
[13]: http://rubydoc.info/gems/pesapal/
[14]: https://github.com/itsmrwave/mo
[15]: https://github.com/itsmrwave/mo/tree/master/convention#-convention
[16]: http://kingori.co/articles/2013/09/modus-operandi/
[17]: https://github.com/itsmrwave/pesapal-gem/pulls
[18]: http://developer.pesapal.com/how-to-integrate/step-by-step
[19]: https://github.com/itsmrwave/pesapal-gem/graphs/contributors
[20]: http://guides.rubyonrails.org/
[21]: https://raw.githubusercontent.com/itsmrwave/pesapal-gem/master/CHANGELOG.md
[22]: https://travis-ci.org/itsmrwave/pesapal-gem/pull_requests
[23]: https://github.com/itsmrwave/pesapal-gem/issues/new
[24]: https://www.pesapal.com/
[25]: http://yardoc.org/
[26]: https://itsmrwave.github.io/pesapal-gem
[27]: http://jekyllrb.com/
[28]: http://jekyllrb.com/docs/home/
[29]: https://github.com/itsmrwave/pesapal-gem/tree/gh-pages
[30]: https://pages.github.com/
[31]: https://relishapp.com/rspec/
[32]: http://en.wikipedia.org/wiki/Code_review
[33]: https://relishapp.com/rspec/docs/gettingstarted
