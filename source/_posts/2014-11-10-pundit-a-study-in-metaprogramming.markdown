---
layout: post
title: "Pundit - A Study in Metaprogramming"
date: 2014-11-10 14:47:38 -0800
comments: true
categories:
---

I know what you're thinking. "Metaprogramming? Why are you using such scary
words?" But it's not as bad as it seems. To keep it simple, metaprogramming
is simply your code writing more code to be decided at runtime. Still confused?
Let's walk through an example to help demonstrate the concept. Fortunately,
there's an excellent gem available which is not only an easy entry into the
topic, but is also widely used - it's worth it to any programmer to understand
how this bit of code works.

# What is Pundit?

Pundit is the single most widely used gem for authorization, and is often paired
with the Devise library for authentication. Pundit is clean and simple,
providing an ideal separation of concerns and putting all of your authorization
logic into a single Plain Old Ruby Object. More importantly, it amounts to
very simple Ruby code that could be implemented by anyone, but provides useful
syntactic sugar to make applying it less unwieldy.

# Jumping in

So how does Pundit work? Essentially, it starts with a class labeled
"#{Model}Policy" which holds all permissions. For example, say we had an app
for use in a hospital which has many users such as admins, doctors, and nurses.
Admins should have a great deal of access, but shouldn't be allowed to see
individual appointments to preserve patient privacy. Nurses and doctors should
be able to see appointments, but only doctors should schedule new ones. Perhaps
less than 100% realistic for an actual hospital, but it gives us a framework to
deal with. For this particular situation, here's what we would expect to see
in our AppointmentPolicy (as well as the ApplicationPolicy it inherits from)

```
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  *Default authorizations set to refuse access

  class Scope

    *Omitted. Scope is handled similarly to authorizations, but we won't
      talk about it here.

  end
end
```

```
class AppointmentPolicy < ApplicationPolicy

  def show?
    user.doctor? || user.nurse?
  end

  def create?
    user.doctor?
  end
end
```

Of course, methods such as .doctor? and .nurse? are defined in the user models,
likely referencing the roles of users within the database.

To invoke these policies for authorization, a programmer could write a direct
reference such as:

```
class AppointmentsController
  def create
    @appointment = Appointment.new
    AppointmentPolicy.new(current_user, @appointment).create?

    if @appointment.save?
      *etc
    end
  end
end
```

Simple, right? You simply reference the class, instantiate it with the currently
signed in user and the record involved in the authorization, and then ask
whether the user in question can execute that particular action on that record.
All of the authorization logic is neatly contained in a simple policy file for
you to change as needed.

Now, we could stop here and call it good. The solution works and it's scalable.
But think about how long some of these calls could get. Here's an example of
a scope call from one of my simpler projects.

```
@comments = CommentPolicy::Scope.new(current_user, @article.comments).resolve
```

Not too complicated here. We're instantiating the Scope sub-class in the
CommentPolicy and passing it the current_user and a scope of comments to be
limited according to the policies listed under the .resolve method. However,
you can see some problems. For one, this particular call isn't very long, but
it still runs to almost 80 characters. Also, if you think about it, a lot of
the information here could be inferred without explicit declaration. Going back
to our original call:

```
AppointmentPolicy.new(current_user, @appointment).create?
```

The first thing we might notice is that the user passed in here can almost
always be expected to be the currently signed in user. Also, the method
'current_user' is used in Devise and very common in other strategies for
authentication. It could be a useful default to simply assume the user for any
authorization should be the result of the current_user method unless otherwise
stated. Also, we could easily infer which policy to use. The record being
passed in is clearly an Appointment, so it would be expected for us to use
the AppointmentPolicy to authorize an action regarding it.

Additionally, if we were to name all our authorization methods after the
controller actions which they are authorizing, we would have a nice parallel
where the create action maps to the create? authorization, the show action
maps to the show? authorization, and so on. Since this is the overwhelmingly
most common use of authorization, why not use that information to build up
our calls?

Put simply, an intelligent human being could easily infer most of the
information above, so why not program a computer to do the same? If we could do
so, we would only have to make short calls in the controller, such as:

```
authorize @appointment
```

Genius! And we find that this is precisely what Pundit does. Now let's start
exploring that authorize! method to see how it does it.

While Pundit has a fair number of files in its source code, we can focus on two,
pundit.rb and policy_finder.rb. pundit.rb holds most of the public methods such
as :authorize or :policy. policy_finder.rb is typically referenced in situations
where the policy is being invoked from a view, as we will soon see. Let's start
from the authorize method in pundit.rb

```
module Pundit
  class NotAuthorizedError < StandardError
    attr_accessor :query, :record, :policy
  end
  class AuthorizationNotPerformedError < StandardError; end
  class PolicyScopingNotPerformedError < AuthorizationNotPerformedError; end
  class NotDefinedError < StandardError; end

  extend ActiveSupport::Concern

  class << self
    def policy_scope(user, scope)
      policy_scope = PolicyFinder.new(scope).scope
      policy_scope.new(user, scope).resolve if policy_scope
    end

    def policy_scope!(user, scope)
      PolicyFinder.new(scope).scope!.new(user, scope).resolve
    end

    def policy(user, record)
      policy = PolicyFinder.new(record).policy
      policy.new(user, record) if policy
    end

    def policy!(user, record)
      PolicyFinder.new(record).policy!.new(user, record)
    end
  end

  included do
    if respond_to?(:helper_method)
      helper_method :policy_scope
      helper_method :policy
      helper_method :pundit_user
    end
    if respond_to?(:hide_action)
      hide_action :policy_scope
      hide_action :policy_scope=
      hide_action :policy
      hide_action :policy=
      hide_action :authorize
      hide_action :verify_authorized
      hide_action :verify_policy_scoped
      hide_action :pundit_user
    end
  end

  def verify_authorized
    raise AuthorizationNotPerformedError unless @_policy_authorized
  end

  def verify_policy_scoped
    raise PolicyScopingNotPerformedError unless @_policy_scoped
  end

  def authorize(record, query=nil)
    query ||= params[:action].to_s + "?"
    @_policy_authorized = true

    policy = policy(record)
    unless policy.public_send(query)
      error = NotAuthorizedError.new("not allowed to #{query} this #{record}")
      error.query, error.record, error.policy = query, record, policy

      raise error
    end

    true
  end

  def policy_scope(scope)
    @_policy_scoped = true
    @policy_scope or Pundit.policy_scope!(pundit_user, scope)
  end
  attr_writer :policy_scope

  def policy(record)
    @_policy or Pundit.policy!(pundit_user, record)
  end

  def policy=(policy)
    @_policy = policy
  end

  def pundit_user
    current_user
  end
end
```

Let's start with the first method that's called - :authorize

```
def authorize(record, query=nil)
  query ||= params[:action].to_s + "?"
  @_policy_authorized = true

  policy = policy(record)
  unless policy.public_send(query)
    error = NotAuthorizedError.new("not allowed to #{query} this #{record}")
    error.query, error.record, error.policy = query, record, policy

    raise error
  end

  true
end
```

We see now that when we call authorize with one argument, the query is nil
and is then assigned to be the string conversion of the action in the params
hash plus a question mark. @_policy_authorized is then set to true and the
:policy= method is called.

```
def policy=(policy)
  @_policy = policy
end
```

Simple enough - combining the call in authorize with the method definition, our
code looks like this:

```
@_policy = policy(record)
```

And here's the policy method:

```
def policy(record)
  @_policy or Pundit.policy!(pundit_user, record)
end
```

We can now fully unpack the authorize line to:

```
@_policy = @_policy or Pundit.policy!(pundit_user, record)
```

Or more familiarly:

```
@_policy ||= Pundit.policy!(pundit_user, record)
```

Since @_policy is not currently defined, we make a call to Pundit.policy! and
pass in our record (@appointment) and the pundit_user

```
def pundit_user
  current_user
end
```

Aha, there's our current_user. Now we see where Pundit is making its assumption
about how to find the currently signed-in user. Let's take a look at
Pundit.policy!

```
def policy!(user, record)
  PolicyFinder.new(record).policy!.new(user, record)
end
```

Well, looks like we're heading over to policy_finder.rb to figure this one out.

```
module Pundit
  class PolicyFinder
    attr_reader :object

    def initialize(object)
      @object = object
    end

    def scope
      policy::Scope if policy
    rescue NameError
      nil
    end

    def policy
      klass = find
      klass = klass.constantize if klass.is_a?(String)
      klass
    rescue NameError
      nil
    end

    def scope!
      scope or raise NotDefinedError, "unable to find scope #{find}::Scope for #{object}"
    end

    def policy!
      policy or raise NotDefinedError, "unable to find policy #{find} for #{object}"
    end

  private

    def find
      if object.respond_to?(:policy_class)
        object.policy_class
      elsif object.class.respond_to?(:policy_class)
        object.class.policy_class
      else
        klass = if object.respond_to?(:model_name)
          object.model_name
        elsif object.class.respond_to?(:model_name)
          object.class.model_name
        elsif object.is_a?(Class)
          object
        elsif object.is_a?(Symbol)
          object.to_s.classify
        else
          object.class
        end
        "#{klass}Policy"
      end
    end
  end
end
```

First let's look at the policy! method:

```
def policy!
  policy or raise NotDefinedError, "unable to find policy #{find} for #{object}"
end
```

First we instantiate the PolicyFinder class with @object being equal to our
record, so now @object == @appointment from our controller. We still don't know
why .new.policy(user, record) is appended after policy!, but let's go through
this step by step and it'll make sense later. Looking at the policy method,
we find that it invokes yet another method and raises a NotDefinedError if
that method returns nil. Let's take a look at that method.

```
def policy
  klass = find
  klass = klass.constantize if klass.is_a?(String)
  klass
rescue NameError
  nil
end
```

We see that a new variable, 'klass', is being set equal to the return of the
find method, after which it is turned into a constant if it is a string, and
then implicitly returned. Nil is returned in the even of a NameError. Simple
enough, let's look at the find method.

```
def find
  if object.respond_to?(:policy_class)
    object.policy_class
  elsif object.class.respond_to?(:policy_class)
    object.class.policy_class
  else
    klass = if object.respond_to?(:model_name)
      object.model_name
    elsif object.class.respond_to?(:model_name)
      object.class.model_name
    elsif object.is_a?(Class)
      object
    elsif object.is_a?(Symbol)
      object.to_s.classify
    else
      object.class
    end
    "#{klass}Policy"
  end
end
```

And here's where we find our metaprogramming magic. Rendered more readably:

```
def find
  if the object has the method :policy_class
    then return that
  else if the class of the object has the method :policy_class
    then return that
  else
    if the object has the method :model_name
      then set klass equal to that
    else if the object's class has the method :model_name
      then set klass equal to that
    else if the object passed in is itself a class
      then set klass equal to that
    else if the object passed in is a symbol
      then set klass equal to a class version of the symbol
    else
      then set klass equal to whatever the class of the object is
    end
    return the concatenation of the klass variable and the string "Policy"
  end
end
```

This find method doesn't know what will be passed in as the record. It could be
a Model class, it could be an instance of a model, it could be a symbol, etc.
It tries a number of methods to figure out what is the appropriate Policy name
for the given record.

Now that the klass variable in :policy has been found (the find method discussed
just now would have returned "AppointmentPolicy"), it's turned into a
constant and returned to the :policy! method.

```
def policy!
  policy or raise NotDefinedError, "unable to find policy #{find} for #{object}"
end
```

We have our policy, so this doesn't raise the error.

Now, since AppointmentPolicy has been turned into a constant rather than its
original string during the find method, we now have a formal class which can
be instantiated. So now our original Pundit.policy! method makes sense

```
def policy!(user, record)
  PolicyFinder.new(record).policy!.new(user, record)
end
```

From this, we now get:

```
AppointmentPolicy.new(user, record)
```

Well, that looks familiar! Remember that the authorize method was supposed to
simplify our original call which was:

```
AppointmentPolicy.new(user, record).create?
```

Let's head all the way back to our authorize method.

```
def authorize(record, query=nil)
  query ||= params[:action].to_s + "?"
  @_policy_authorized = true

  policy = policy(record)
  unless policy.public_send(query)
    error = NotAuthorizedError.new("not allowed to #{query} this #{record}")
    error.query, error.record, error.policy = query, record, policy

    raise error
  end

  true
end
```

Following our way back up the chain of method calls, we get back to our original
line:

```
policy = policy(record)
```

Which in our example now unpacks to:

```
@_policy = AppointmentPolicy.new(current_user, @appointment)
```

The next line simply calls a public method on @_policy, and that method is the
query as determined earlier in the method. In other words:

```
unless AppointmentPolicy.new(current_user, @appointment).create?
```

And there we have it. If this create? method returns false, a
NotAuthorizedError will be raised, halting the controller action being
authorized. If create? returns true, then the authorize method returns true and
the controller action continues uninterrupted.

# Conclusion

As you can see, this is an immense improvement over our original call. With just
a few sensible assumptions about how the method is called, it's possible to
vastly simplify our code. Metaprogramming often seems like magic, but like any
code, it's actually quite straightforward. Don't be afraid to look into the
source code of something that seems like magic. It's often easier than you
think, and who knows? You just might learn something. I know I did.
