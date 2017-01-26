---
layout: documentation
title: Safe
---

In MongoDB, database writes/updates follow the "fire and forget" pattern. This means that the database will not wait for a response when you perform an update operation on a document. MongoMapper follows this same convention by default.

The down side of this strategy is if something goes wrong–such as a unique index constraint failing or an internal MongoDB error–an error won't be raised. You may never know that your data isn't being saved, until you try to read it and it's not there.

Instead of just using the default "fire and forget" behavior on our save operation, passing the `:safe` option to `save` will force the driver to make sure the save succeeds and raise an error if it doesn't.

{% highlight ruby %}
user = User.new("duplicate@example.com")
user.save(:safe => true) # => Mongo::OperationFailure: 11000: E11000 duplicate key error index: mealsnap_dev.foos.$email_1  dup key: { : "duplicate@example.com@" }
{% endhighlight %}

You could also declare `safe` in your model in order to force all operations to be safe.

{% highlight ruby %}
class User
  include MongoMapper::Document
  safe

  key :email, String
end
{% endhighlight %}

The `safe` option can be either a boolean or a Hash of options. For example, if you want to ensure that a highly critical write is committed to the journal on the majority of replica set members before `save` returns, you would use:

{% highlight ruby %}
invoice = Invoice.new(:status => :paid)
invoice.save(:safe => {:j => true, :w => 'majority'})
{% endhighlight %}

You can also set these options when you declare your model:

{% highlight ruby %}
class Invoice
  include MongoMapper::Document
  safe(:j => true, :w => 'majority')
end
{% endhighlight %}

For more information about the available options, see the documentation for the [getLastError command](http://www.mongodb.org/display/DOCS/getLastError+Command).