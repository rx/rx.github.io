---
layout: post
title: Rails exception handling - Egregious
date: '2011-09-27T09:45:00.000-07:00'
author: Rx
tags:
modified_time: '2015-10-23T08:40:54.210-07:00'
blogger_id: tag:blogger.com,1999:blog-3950436325253663978.post-5977641490449370728
blogger_orig_url: http://railsindirection.blogspot.com/2011/09/rails-exception-handling-egregious.html
---

- Version 0.2.10 released to [rubygems](https://rubygems.org/gems/egregious) on 10.27.2015
  - Added support for exceptions that define a http\_status.   
    The exception map can override this.  
    This is a good way to allow a raise to specify a http\_status using a custom exception.  
    The idea for this came from the Stripe::Error exception classes.  


Egregious is a rails based exception handling gem for well defined http exception handling for
json, xml and html.

If you have a json or xml api into your rails application, you probably
have added your own exception handling to map exceptions to a http status and formatting your json and xml output.


You probably have code sprinkled about like this:

{% highlight ruby %}
rescue_from CanCan::AccessDenied do |exception|
 flash[:alert] = exception.message
 respond_to do |format|
 format.html { redirect_to dashboard_path }
 format.xml {
      render :xml => exception.to_xml, :status => :forbidden }
 format.json { render :json=>
      exception.to_json, :status => :forbidden }
 end
end
{% endhighlight %}

This example is straight from the CanCan docs. You'll notice a couple of things here. This handles the
CanCan::AccessDenied exception only. It then will redirect to the startup page, or render xml and json returning
the http status code of :forbidden (403). You can see one of the first features of the Egregious gem. We extend
Exception to add the to_xml and to_json methods. These return a well structured error that can be consumed by the
API client.
{% highlight ruby %}
Exception.new("Hi Mom").to_xml
{% endhighlight %}
returns:
{% highlight xml %}
<errors>
    <error>Hi Mom</error>
    <type>Exception</type>
</errors>
{% endhighlight %}
{% highlight ruby %}
Exception.new("Hi Dad").to_json
{% endhighlight %}
returns:
{% highlight json %}
{"error":"Hi Dad", "type":"Exception"}
{% endhighlight %}

So that's pretty handy in itself. Now all exceptions have a json and xml api that describe them. It happens to be the same
xml and json that is returned from the errors active record object, with the addition of the type element. That
allows you to mix and match validations and exceptions. Wow, big deal. We'll it is. If you are writing a client
then you need to have a very well defined error handling. I'd like to see all of rails do this by default. So that
anyone interacting with a rails resource has a consistent error handling experience. (Expect more on being a good
REST API in future posts.) As a client we can now handle errors in a consistent way.

Besides the error message we would like a well defined mapping of classes of exceptions to http status codes. The idea is
that if I get back a specific http status code then I can program against that 'class' of problems. For example if
I know that what I did was because of invalid input from my user, I can display that message back to the user.
They can correct it and continue down the path. But if the Http status code says that it was a problem with the
server, then I know that I need to log it and notify someone to see how to resolve it.

We handle all exceptions of a given class with a mapping to an http status code. With all the most common Ruby,
Rails, Devise, Warden and CanCan exceptions having reasonable defaults. (Devise, Warden and CanCan are all
optional and ignored if their gems are not installed.)

As of 0.2.9 you can also define a
method named 'http_status' on the exception and it will be used as the status code. This is a nice pattern that
allows you to raise an exception and specify the status code. The Egregious::Error allows you to do this as a
second parameter to initialize:

{% highlight ruby %}
    raise Egregious::Error.new("My very bad error", :payment_required)
{% endhighlight %}


 If the problem
was the api caller then the result codes are in the 300 range. If the problem was on the server then the status
codes are in the 500 range. 

I'm guessing if you bother to read this far, you are probably
interested in using Egregious. Its simple to use and configure. To install:

In you Gemfile
add the following:

{% highlight ruby %}
gem 'egregious'
{% endhighlight %}



In
your ApplicationController class add the following at or near the top:

{% highlight ruby %}
class ApplicationController < ActionController::Base
    include Egregious
    # your code ...
end
{% endhighlight %}



That's it. You will now get
reasonable api error handling.

If you want to add your own exceptions to http status codes
    end
mappings, or change the defaults add an initializer and put the following into it:


{% highlight ruby %}
Egregious.exception_codes.merge!({NameError => :bad_request})
{% endhighlight %}


Here
you can re-map anything and you can add new mappings. 

Note: If you think the default
exception mappings should be different, please contact me via the <a
        href="https://github.com/voomify/egregious">Egregious github project</a>.

We also
created exceptions for each of the http status codes, so that you can throw those exceptions in your code. Its an
easy way to throw the right status code and setup a good message for it. If you want to provide more context, you
can derive you own exceptions and add mappings for them.

Here is an example of throwing a
bad request exception: 

{% highlight ruby %}
raise Egregious::BadRequest.new("You can not created an order without a customer.") unless customer_id
{% endhighlight %}



Egregious adds
mapping of many exceptions, if you have your own rescue_from handlers those will get invoked. You will not lose
any existing behavior, but you also might not see the changes you expect until you remove or modify those
rescue_from calls. At a minimum I suggest using the .to_xml and .to_json calls io your existing rescue_from
methods/blocks.

And finally if you don't like the default behavior. You can override any
portion of it and change it to meet your needs.

If you want to change the behavior then you
can override the following methods in your ApplicationController.

{% highlight ruby %}
# override this if you want your flash to behave differently
def egregious_flash(exception)
    flash.now[:alert] = exception.message
end
{% endhighlight %}





{% highlight ruby %}
# override this if you want your logging to behave differently
def
      egregious_log(exception)
 logger.fatal(
 "\n\n" + exception.class.to_s + ' (' +
      exception.message.to_s + '):\n ' +
 clean_backtrace(exception).join("\n ") +
 "\n\n")
 HoptoadNotifier.notify(exception) if defined?(HoptoadNotifier)
end
{% endhighlight %}

<br
        />
{% highlight ruby %}
# override this if you want to change your respond_to behavior
def egregious_respond_to(exception)
 respond_to do |format|
    status = status_code_for_exception(exception)
    format.xml { render :xml=> exception.to_xml, :status => status }
    format.json { render :json=> exception.to_json, :status => status }
      # render the html page for the status we are returning it exists...if not then render the 500.html page.
    format.html { render :file => File.exists?(build_html_file_path(status)) ?
    build_html_file_path(status) : build_html_file_path('500')}
 end
end
{% endhighlight %}

<br
        />
{% highlight ruby %}
# override this if you want to change what html static file gets returned.
def build_html_file_path(status)
    File.expand_path(Rails.root, 'public', status + '.html')
end
{% endhighlight %}



{% highlight ruby %}
# override this if you want to control what gets sent to airbrake
# optionally you can configure the airbrake ignore list
def notify_airbrake(exception)
 # tested with airbrake 3.1.15 and 4.2.1
 env['airbrake.error_id'] = Airbrake.notify_or_ignore(exception) if defined?(Airbrake)
end
{% endhighlight %}



We are using this gem in all our Rails projects.

Go forth and be egregious!
