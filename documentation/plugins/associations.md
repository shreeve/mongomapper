---
layout: documentation
title: Associations
---

MongoMapper allows you to define the relationship between your models. When you define an association, MongoMapper creates methods on your model that make it easy to create, break, find, and persist the connections between models in your application.

-   [Defining Associations](#defining)
    -   [One-to-Many](#one-to-many)
    -   [Many-to-Many](#many-to-many)
    -   [One-to-One](#one-to-one)
-   [Embedded Documents](#embedded)
-   [Polymorphic Associations](#polymorphic)
    -   [Basic Polymorphism](#basic-polymorphism)
    -   [Polymorphism with SCI](#polymorphism-with-sci)
    -   [Embedded Polymorphism](#embedded-polymorphism)
-   [Association Extensions](#extensions)

Defining Associations
---------------------

Associations between models in MongoMapper are defined by combining the [`many`](#many), [`belongs_to`](#belongs_to), and [`one`](#one) methods. These three methods can form a one-to-many, many-to-many, or one-to-one association.

### One-to-Many

Use [`many`](#many) and [`belongs_to`](#belongs_to) in your models to form a one-to-many association.

{% highlight ruby %}
class Tree
  include MongoMapper::Document
  many :birds
end

class Bird
  include MongoMapper::Document
  belongs_to :tree
end
{% endhighlight %}

In this one-to-many relationship, one tree can have many birds perching on it, and a bird can only be in one tree at a time.

### Many-to-Many

In a relational database, you might model a many-to-many relationship by creating a "join table." MongoDB doesn't have joins. But because arrays are first class citizens in MongoDB, you can simply store an array of ObjectId's.

Use an array key and [`many`](#many) with the `:in` option in your model to form a many-to-many association.

{% highlight ruby %}
class Book
  include MongoMapper::Document
  key :title
  key :author_ids, Array
  many :authors, :in => :author_ids
end

class Author
  include MongoMapper::Document
  key :name
end
{% endhighlight %}

Each book stores an array of the id's of its authors in the `author_ids` key. This allows books to have many authors, and authors to have many books.

Currently, many-to-many associations are one-sided in MongoMapper. [Support will be added](https://github.com/mongomapper/mongomapper/issues/240) in the near future for the inverse.

### One-to-One

Use [`one`](#one) and [`belongs_to`](#belongs_to) in your models to form a one-to-one association.

{% highlight ruby %}
class Employee
  include MongoMapper::Document
  key :name
  one :desk
end

class Desk
  include MongoMapper::Document
  key :color
  belongs_to :employee
end
{% endhighlight %}

In this one-to-one relations, an employee can only use one desk, and each desk can only be used by one employee.

Embedded Documents
------------------

When one document will almost always be fetched with another, it makes sense to just embed it into the same document. [`many`](#many) and [`one`](#one) associations can be used with embedded models. See [EmbeddedDocument](/documentation/embedded-document.html) for more information.

Polymorphic Associations
------------------------

MongoMapper lets you define *polymorphic* associations where one side of the association won't always be the same class. The polymorphism can be done either the [basic way](#basic-polymorphism) (polymorphic documents in many different collections) or using [single collection inheritance](#polymorphism-with-sci) (polymorphic documents in one collection). See the examples below.

**Note:** All of these polymorphic examples use the `many` association, but MongoMapper's polymorphism works fine with `one` in place of `many`.

### Basic Polymorphism

Say you want users to be able to comment on articles *and* products:

{% highlight ruby %}
class Article
  include MongoMapper::Document
  key :title
  many :comments, :as => :commentable
end

class Product
  include MongoMapper::Document
  key :sku
  many :comments, :as => :commentable
end

class Comment
  include MongoMapper::Document
  key :text
  belongs_to :commentable, :polymorphic => true
end
{% endhighlight %}

In the database, MongoMapper adds an extra `commentable_type` key so that it can fetch each comment's parent from the proper collection.

In the database, a comment might look like this when converted to Ruby:

{% highlight ruby %}
{
  "_id"  => BSON::ObjectId('...'),
  "text" => "It broke after a month.  But it was a good month.",
  "commentable_type" => "Product",
  "commentable_id"   => BSON::ObjectId('...')
}
{% endhighlight %}

### Polymorphism with SCI

Polymorphism can also be provided by [Single Collection Inheritance](/documentation/plugins/single-collection-inheritance.html). The previous example could be rewritten:

{% highlight ruby %}
class Commentable
  include MongoMapper::Document
  many :comments, :as => :commentable
end

class Article < Commentable
  key :title
end

class Product < Commentable
  key :sku
end

class Comment
  include MongoMapper::Document
  key :text
  belongs_to :commentable
end
{% endhighlight %}

Though you can't have polymorphism on the `belongs_to` side with the [basic method](#basic-polymorphism), with SCI you can:

{% highlight ruby %}
class Site
  include MongoMapper::Document
  many :pages
end

class Page
  include MongoMapper::Document
  belongs_to :site
end

class HomePage < Page
  key :content
end

class BlogPost < Page
  key :title
  key :body
end
{% endhighlight %}

And SCI also allows [many-to-many](#many-to-many) polymorphism, which also can't be done the [basic way](#basic-polymorphism). For example, here's a richer model of books and authors:

{% highlight ruby %}
class Book
  include MongoMapper::Document
  key :title
  key :author_ids, Array
  many :authors, :in => :author_ids
end

class Novel < Book
  key :genre
end

class Textbook < Book
  key :subject
end

class Author
  include MongoMapper::Document
  key :name
end

class Editor < Author
  key :company_name
end
{% endhighlight %}

Single collection inheritance is needed for polymorphism on the `belongs_to` side and for many-to-many polymorphism because when MongoMapper looks for associated documents, it only fires one query on one collection&mdash;therefore all of an object's children must be located in that one collection. On the other hand, with [basic polymorphism](#basic-polymorphism) the different document types are located in different collections.

### Embedded Polymorphism

[Embedded documents](/documentation/embedded-document.html) may also be polymorphic. Both [basic polymorphism](#basic-polymorphism) and [SCI polymorphism](#polymorphism-with-sci) work fine for all embedded associations. Here's one example:

{% highlight ruby %}
class Human
  include MongoMapper::Document
  key :name
  many :contact_methods, :polymorphic => true
end

class ContactMethod
  include MongoMapper::EmbeddedDocument
  key :name
  embedded_in :human
end

class Email < ContactMethod
  key :email
end

# basic polymorphism: it doesn't have to inherit
class Address
  include MongoMapper::EmbeddedDocument
  key :street_address
  key :city
  embedded_in :human
end
{% endhighlight %}

MongoMapper stores the class name in a `_type` key so the objects get loaded as the proper class. For example, a human in the database might look like this when converted to Ruby:

{% highlight ruby %}
{
  "_id"  => BSON::ObjectId('...'),
  "name" => "Maria"
  "contact_methods" => [
    {
      "_id"   => BSON::ObjectId('...'),
      "_type" => "Email",
      "email" => "mariamusic982452@gmail.com"
    },
    {
      "_id"   => BSON::ObjectId('...'),
      "_type" => "Address",
      "street_address" => "123 Fun St.",
      "city"  => "Nowhere, MI"
    }
  ]
}
{% endhighlight %}

Association Extensions
----------------------

You can extend your core model associations to help you return variations on the associated objects, or encapsulate behavior related to the association.

{% highlight ruby %}
class Blog
  include MongoMapper::Document

  many :posts do
    def published
      where(:published => true)
    end
  end
end

@blog = Blog.first
@blog.posts.published
{% endhighlight %}

If you want reuse an extension between many associations, define a module.

{% highlight ruby %}
module Publishable
  def published
    where(:published => true)
  end

  def publish!
    set(:published => true)
  end
end

class Blog
  include MongoMapper::Document

  many :posts, :extend => Publishable
  many :comments, :extend => Publishable
end
{% endhighlight %}

Sometimes you need access to the proxy owner or target objects in your extension. The following methods are available inside of your extension:

-   proxy\_owner - the object the association is part of.
-   proxy\_target - the target of the association, either a single object for a belongs\_to or one association, or an array for a many association.
-   proxy\_association - the object that describes the association.