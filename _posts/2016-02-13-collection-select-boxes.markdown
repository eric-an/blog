---
layout: post
title:  "Checkboxes in forms for Rails"
date:   2016-02-13
permalink: /collection_check_boxes
---

Imagine we are creating a web application to help a conference coordinator organize and confirm which attendees will attend which sessions. This can be completed in Rails using collection select boxes.

**Defining the Models**
---

Let's first create the models and associations between `sessions` and `attendees`.

{% highlight ruby linenos%}
class Session < ActiveRecord::Base
  has_many :session_attendees
  has_many :attendees, through: :session_attendees
end
{% endhighlight %}

{% highlight ruby linenos%}
class Attendee < ActiveRecord::Base
  has_many :session_attendees
  has_many :sessions, through: :session_attendees
end
{% endhighlight %}

We'll next create the join model that associates `sessions` with `attendees`.

{% highlight ruby linenos%}
class SessionAttendee < ActiveRecord::Base
  belongs_to :session
  belongs_to :attendee
end
{% endhighlight %}

We can know establish that each session has many attendees and every attendee can have many sessions.

**Generate the Form**
---

Our coordinator wants to view a page that will create a new session and record all the attendees for that particular session. Assuming that the attendees' information has already been entered into the database, we will list all of them on the page through checkboxes using the `form_for` helper method to create a form.

{% highlight ruby linenos%}
<%= form_for @session do |f| %>
  <%= f.label :name %>
  <%= f.text_field :name %>
  <%= f.label :scheduled_time %>
  <%= f.text_area :scheduled_time %>
  <%= f.collection_check_boxes :attendee_ids, Attendee.all, :id, :name %>
  <%= f.submit %>
<% end %>
{% endhighlight %}

Let's examine the `colleciton_check_boxes` method a little closer and the params that we are passing into it.

**`:attendee_ids`**: the collection of all `attendee_ids` that will be passed in if the particular checkbox is marked

**`Attendee.all`**: the collection of all possible checkbox options that will be displayed on the form

**`:id`**: the parameter of an attendee that will be passed into the `attendee_ids` collection

**`:name`**: the attribute of the attendee that will be visible next to the checkbox

The form will be rendered in HTML as:

{% highlight HTML linenos%}
<input type="checkbox" value="1" name="session[attendee_ids][]" id="session_attendee_ids_1" />
<label for="session_attendee_ids_1">Mr. Peabody</label>
<input type="checkbox" value="2" name="session[attendee_ids][]" id="session_attendee_ids_2" />
<label for="session_attendee_ids_2">Sherman</label>
<input type="hidden" name="session[attendee_ids][]" value="" />
{% endhighlight %}

Notice that a hidden input field is created, so that all the checkboxes can be left unchecked. After clicking submit, the `params` will be passed to the controller as a nested hash:

{% highlight ruby %}
{:session => {:attendee_ids => ["1", "2", ""]}} 
{% endhighlight %}

Which brings us to the final step.


**Whitelist the `params` in the Controller**
---

To prevent [CSRF](http://guides.rubyonrails.org/security.html#cross-site-request-forgery-csrf){:target="_blank"} attacks, we will add `attendee_ids` to the permitted params. Note: to permit a params attribute defined with an array, we must pass it in with an empty array: 

{% highlight ruby %}
params.require(:session).permit(:name, :scheduled_time, attendee_ids:[])
{% endhighlight %}

Our convention coordinator is now satisfied with a form that will create a new session and record all attendees who are well, attending.