---
layout: documentation
title: Validations
---

MongoMapper uses [ActiveModel:Validations](http://api.rubyonrails.org/classes/ActiveModel/Validations.html), so it works almost exactly like [ActiveRecord](http://api.rubyonrails.org/classes/ActiveModel/Validations/HelperMethods.html).

{% highlight ruby %}
class Person
  include MongoMapper::Document

  key :first_name,  String
  key :last_name,   String
  key :age,         Integer
  key :born_at,     Time
  key :active,      Boolean
  key :fav_colors,  Array

  validates_presence_of :first_name
  validates_presence_of :last_name
  validates_numericality_of :age

  validate :custom_validation

  def custom_validation
    if age < 20
      errors.add( :age, "Youth is wasted on the young")
    end
  end
end
{% endhighlight %}

Validations are run when attempting to save a record. If validations fail, save will return `false`.

{% highlight ruby %}
@person = Person.new(:first_name => "Jon", :last_name => "Kern", :age => 9)
if @person.save
  puts "#{p.first_name} #{p.last_name} (age #{p.age})"
else
  puts "Error(s): ", p.errors.map {|k,v| "#{k}: #{v}"}
end
# Error(s):
# age: Youth is wasted on the young
{% endhighlight %}

Shorthand
---------

Most simple validations can be declared along with the keys.

{% highlight ruby %}
class Person
  include MongoMapper::Document

  key :first_name,  String,   :required => true
  key :last_name,   String,   :required => true
  key :age,         Integer,  :numeric => true
  key :born_at,     Time
  key :active,      Boolean
  key :fav_colors,  Array
end
{% endhighlight %}

The available options when defining keys are:

-   **:required** - Boolean that declares `validates_presence_of`
-   **:unique** - Boolean that declares `validates_uniqueness_of`
-   **:numeric** - Boolean that declares `validates_numericality_of`
-   **:format** - Regexp that is passed to `validates_format_of`
-   **:in** - Array that is passed to `validates_inclusion_of`
-   **:not\_in** - Array that is passed to `validates_exclusion_of`
-   **:length** - Integer, Range, or Hash that is passed to `validates_length_of`