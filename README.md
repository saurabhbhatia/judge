# Judge

[![Build status](https://secure.travis-ci.org/joecorcoran/judge.png?branch=master)](http://travis-ci.org/joecorcoran/judge)

Judge allows easy client side form validation for Rails 3 by porting many `ActiveModel::Validation` features to JavaScript. The most common validations work through JSON strings stored within HTML5 data attributes and are executed purely on the client side. Wherever you need to, Judge provides a simple interface for AJAX validations too.

## Rationale

Whenever we need to give the user instant feedback on their form data, it's common to write some JavaScript to test input values. But since the integrity of our data is handled on the server side, whatever we write in Ruby has to be copied as closely as possible in JavaScript and we end up with some very unsatisfying duplication of application logic.

It would be better to safely expose the validation information from our models to the client – this is where Judge steps in.

## Installation

Judge only supports Rails 3 and from version 2.0.0 onwards, you'll need Rails 3.1 or higher.

Judge relies on [Underscore.js](underscore) in general and [JSON2.js](json2) for browsers that lack proper JSON support. If your application already makes use of these files, you can safely ignore the versions provided with Judge.

### With asset pipeline enabled (Rails >= 3.1)

Add `judge` to your Gemfile and run `bundle install`. That's it! If you need to be explicit in your asset manifest then something like this will do:

```
//= require underscore
//= require json2
//= require judge
```

### Without asset pipeline

Add `judge` to your Gemfile and run `bundle install`. Then run

    $ rails generate judge:install path/to/your/js/dir

to copy judge.js to your application. There are options to copy the dependencies too.

    $ rails generate judge:install --json2 --underscore
    
## Getting started

Add a simple validation to your model.

```ruby
class Post < ActiveRecord::Base
  validates :title, :presence => true
end
```

Make sure your form uses the Judge::FormBuilder and add the :validate option to the field.

```ruby
form_for(@post, :builder => Judge::FormBuilder) do |f|
  f.text_field :title, :validate => true
end
```

On the client side, you can now validate the title input.

```javascript
judge.validate(document.getElementById('post_title'),
  function(element) {
    // valid
    element.style.border = '1px solid green';
  },
  function(element, messages) {
    // invalid
    element.style.border = '1px solid red';
    alert(messages.join(','));
  }
);
```

## Judge::FormBuilder

You can use any of the methods from the standard ActionView::Helpers::FormBuilder – just add :validate => true to the options hash.

```ruby
f.date_select :birthday, :validate => true
```

If you need to use Judge in conjunction with your own custom `FormBuilder` methods, make sure your `FormBuilder` inherits from `Judge::FormBuilder` and use the `#add_validate_attr!` helper.

```ruby
class MyFormBuilder < Judge::FormBuilder
  def fancy_text_field(method, options = {})
    add_validate_attr!(self.object, method, options)
    # do your stuff here
  end
end
```

## Available validators

* presence;
* length (options: *minimum*, *maximum*, *is*);
* exclusion (options: *in*);
* inclusion (options: *in*);
* format (options: *with*, *without*); and
* numericality (options: *greater_than*, *greater_than_or_equal_to*, *less_than*, *less_than_or_equal_to*, *equal_to*, *odd*, *even*, *only_integer*);
* acceptance;
* confirmation (input and confirmation input must have matching ids);
* uniqueness;
* any `EachValidator` that you have written, provided you add a JavaScript version too and add it to `judge.customValidators`.

Options like *if*, *unless* and *on* are not available as they relate to record persistence.

The *allow_blank* option is available everywhere it should be. Error messages are looked up according to the [Rails i18n API](http://guides.rubyonrails.org/i18n.html#translations-for-active-record-models).

## Validating uniqueness

Since validating a record's uniqueness requires an AJAX request, you need to take the extra step of mounting the Judge engine to your application. Do this in your *config/routes.rb* file:

```ruby
Rails.application.routes.draw do
  mount Judge::Engine => '/judge'
end
```

You can choose a path other than `'/judge'` if you need to; just make sure to set this on the client side too:

```javascript
judge.enginePath = '/whatever';
```

## Writing your own validators

If you create a custom `EachValidator`, Judge provides a way to ensure that your I18n error messages are available on the client side. Simply pass to `uses_messages` any number of message keys and Judge will look up the translated messages. Let's run through an example.

```ruby
class FooValidator < ActiveModel::EachValidator
  uses_messages :not_foo

  def validate_each(record, attribute, value)
    unless value == 'foo'
      record.errors.add(:title, :not_foo)
    end
  end
end
```

We'll use the validator in the example above to validate the title attribute of a Post object:

```ruby
class Post < ActiveRecord::Base
  validates :title, :foo => true
end
```

```ruby
form_for(@post, :builder => Judge::FormBuilder) do |f|
  text_field :title, :validate => true
end
```

Judge will look for the `not_foo` message at
*activerecord.errors.models.post.attributes.title.not_foo*
first and then onwards down the [Rails I18n lookup chain](http://guides.rubyonrails.org/i18n.html#translations-for-active-record-models).

Our client side validator would then look something like this:

```javascript
judge.customValidators.foo = function(options, messages) {
  var errorMessages = [];
  // 'this' refers to the DOM element
  if (this.value !== 'foo') {
    errorMessages.push(messages.not_foo);
  }
  return new judge.Validation(errorMessages);
};
```

All client side validators must return a `Validation` – an object that can exist in three different states: *valid*, *invalid* or *pending*. If your validator function is synchronous, you can return a closed `Validation` simply by passing an array of error messages to the constructor.

```javascript
new judge.Validation([]);
  // => empty array, this Validation is 'valid'
new judge.Validation(['must not be blank']);
  // => array has messages, this Validation is 'invalid'
```

The *pending* state is provided for asynchronous validation; a `Validation` object we will close some time in the future. Let's look at an example, using jQuery's popular `ajax` function:

```javascript
judge.customValidators.bar = function() {
  var validation = new judge.Validation();
  $.ajax('/bar-checking-service').done(function(messages) {
    // 'messages' is a JSON array of error messages
    // You can close a Validation with either an array
    // or an unparsed JSON array
    validation.close(messages);
  });
  return validation;
};
```

In the unlikely event that you don't already use a library with AJAX capability, a basic function is provided for making GET requests as follows:

```javascript
judge.get('/something',
  function(status, headers, text) {
    // success (status code 20x)
  },
  function(status, headers, text) {
    // error (any other status code)
  }
);
```


## Judge extensions

If you use [Formtastic](https://github.com/justinfrench/formtastic) or [SimpleForm](https://github.com/plataformatec/simple_form), there are extension gems to help you use Judge within your forms without any extra setup. They are essentially basic patches that add the `:validate => true` option to the `input` method.

### Formtastic

https://github.com/joecorcoran/judge-formtastic

```ruby
gem 'judge-formtastic', '~> x.x.x', :require => 'judge/formtastic'
```

```ruby
semantic_form_for(@user) do |f|
  f.input :name, :validate => true
end
```

### SimpleForm

https://github.com/joecorcoran/judge-simple_form

```ruby
gem 'judge-simple_form', '~> x.x.x', :require => 'judge/simple_form'
```

```
simple_form_for(@user) do |f|
  f.input :name, :validate => true
end
```

### License

Released under an MIT license (see LICENSE.txt).

[blog.joecorcoran.co.uk](http://blog.joecorcoran.co.uk) | [@josephcorcoran](http://twitter.com/josephcorcoran)