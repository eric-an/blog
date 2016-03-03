---
layout: post
title:  "Creating a CRUD app"
date:   2016-02-05
permalink: /fundmycause
---

A full-fledged CRUD application that utilizes RESTful routes is a major undertaking made easier with Ruby frameworks like Sinatra. This domain model creates users with authentication capabilities, so they can create causes that are categorized. Let's walk through the process to see what composes a CRUD app.

**File Structure**
---

An overview of the necessary files:

    └── fundmycause
        ├── Gemfile
        ├── Gemfile.lock
        ├── README.md
        ├── Rakefile
        ├── app
        │   ├── controllers
        │   │   ├── application_controller.rb
        │   │   ├── categories_controller.rb
        │   │   ├── causes_controller.rb
        │   │   └── users_controller.rb
        │   ├── models
        │   │   ├── category.rb
        │   │   ├── cause.rb
        │   │   ├── user.rb
        │   │   └── usercause.rb
        │   └── views
        │       ├── categories
        │       │   └── index.erb
        │       ├── causes
        │       │   ├── edit.erb
        │       │   ├── index.erb
        │       │   ├── new.erb
        │       │   └── show.erb
        │       ├── index.erb
        │       ├── layout.erb
        │       └── users
        │           ├── login.erb
        │           ├── new.erb
        │           └── show.erb
        ├── config
        │   └── environment.rb
        ├── config.ru
        ├── db
        │   └── schema.rb
        └── public
            └── stylesheets
                └── index.css

**Gemfile**
---

Explore the `Gemfile` to see the relevant gems that this gem will require. Particular to this app is the `bcrypt` gem. This will store the user-created password by 'salting' in random characters. 


**Models**
---

This app creates three models: a `User`, a `Cause`, and a `Category`. In creating the migrations, note the following associations:

    - a `User` has many `Causes`
    - a `Cause` has many `Users`
    - a `Cause` belongs to a `Category`
    - a `Category` has many `Causes`


**User Authentication**
---

To add authentication to the app, the two helper methods `logged_in?` and `current_user` will be utilized to check user credentials before accessing certain pages. These methods will prevent users from editing and deleting other users' causes as well as restricting access to only their own causes.


**CRUD: Create a Cause**
---
{% highlight ruby linenos%}
class CausesController < ApplicationController

  get '/causes/new' do
    if logged_in?
      erb :"causes/new"
    else
      redirect to '/login'
    end
  end

  post '/causes/new' do
    @user = current_user
    @cause = Cause.new(name: params[:cause][:name])
    @cause.created_by_user = @user.id
    @cause.description = params[:cause][:description]
    @cause.funding = params[:cause][:funding]
    if params[:cause][:category_id]
      @cause.category_id = params[:cause][:category_id]
    else
      @cause.category = Category.find_or_create_by(name: params[:new_category].capitalize)
    end

    if @cause.save
      @cause.users << @user if !@cause.users.include?(@user)
      redirect to "/causes"
    else
      erb :"causes/new", locals: {message: "The cause wasn't created."}
    end
  end
end
{% endhighlight %}

The first few routes will `create` the form and process the form input to create a new `Cause`. The different attributes of the `Cause` (like name, description, created by the user, and funding) will be defined with values from the form submission. The user will also have an option to choose from an existing `Category` or create a new `Category` to associate with the `Cause`.

To accomplish this, the `POST` method on the create action will utilize a conditional statement to check if a particular category checkbox has been marked. If not, a new `Category` will be created and associated with the new `Cause` with the value from the form input for a new category.

Finally, if the new `Cause` can be saved to the database, the `current_user` will be associated with the `Cause` so that the "creator" of the cause will be defined and makes it easier to formulate the restricted edit and delete actions.

**CRUD: Read a Cause**
---
{% highlight ruby linenos%}
get '/causes' do
  @causes = Cause.all
  @categories = Category.all
  erb :"causes/index"
end

get '/cause/:id' do
  @cause = Cause.find_by(id: params[:id])
  @user = current_user if logged_in?
  erb :"causes/show"
end
{% endhighlight %}

In order to view a list of all causes and detailed information on each cause, the `index` and `show` routes and view pages must be created. On the `show` page, there is a conditional that will be implemented to check if the user is `logged_in?` and whether or not the cause's first user is equal to the `current_user`. 

In essence, this checks to see if the cause's creator is the current user. If it is, it will present the user with an option to delete the cause. If not, it will have an option for the user to join the cause. In order to see the routes and code associated with joining/removing a user from a cause, view the code at [fundmycause](https://github.com/eric-an/fundmycause/blob/master/app/controllers/causes_controller.rb#L42-L51){:target="_blank"}.


**CRUD: Edit a Cause**
---
{% highlight ruby linenos%}
get '/cause/:id/edit' do
  if logged_in?
    @user = current_user
    @cause = Cause.find_by(id: params[:id])
    if @cause.created_by_user == @user.id
      erb :"causes/edit"
    else
      redirect to '/causes'
    end
  else
    redirect to '/causes'
  end
end

patch '/cause/:id/edit' do
  @cause = Cause.find_by(id: params[:id])
  @cause.name = params[:cause][:name]
  @cause.description = params[:cause][:description]
  @cause.funding = params[:cause][:funding]
  if params[:cause][:category_id]
    @cause.category_id = params[:cause][:category_id]
  else
    @cause.category = Category.find_or_create_by(name: params[:new_category].capitalize)
  end

  if @cause.save
    redirect to "/cause/#{@cause.id}"
  else
    erb :"causes/show", locals: {message: "The cause wasn't updated."}
  end
end
{% endhighlight %}

Before the `edit` cause page is loaded, the user will be authenticated with the `logged_in?` helper method. After authentication, there will be a conditional statement to check if the cause's first user (a.k.a. the creator) is the `current_user`. If so, the page for editing a `Cause` will load and produce a form simliar to the "create a new cause" view page.

Upon form submission, the information stored in the params hash will be used to update the `cause`. Similar to the create action, the PATCH method on the edit action invokes a conditional statement that potentially assigns a different category - by choosing from available ones or creating a new one.

A note on the PATCH method: By utilizing the `MethodOverride` class available through `Rack`, the HTTP POST request method is overridden by the value of the `_method` parameter located in the form. 

**CRUD: Delete a Cause**
---
{% highlight ruby linenos%}
delete '/cause/:id/delete' do
  if logged_in?
    @user = current_user
    @cause = Cause.find_by(id: params[:id])
    if @cause.created_by_user == @user.id
      @cause.destroy
      erb :"users/show", locals: {message: "The cause was deleted."}
    else
      redirect to '/causes'
    end
  else
    redirect to '/causes'
  end
end
{% endhighlight %}

Similar to the `edit` action, the `delete` action will authenticate the user and confirm with a conditional statement that the cause's first user (the `cause` creator) is the `current_user`. With that authentication, the `delete` action will display as a button on the show view page and successfully delete the `cause`.

Also similar to the `edit` action, the `MethodOverride` class is utilized to turn the POST request into a DELETE request.

**Summary**
---

This domain model utilized the MVC framework and CRUD actions to develop routes and view pages that associate `Users` with `Causes` and `Causes` with `Categories`. By examining each component of a CRUD application, we are able to see how RESTful routes are beneficial and efficient in allowing easy data creation and manipulation.

See the entire project on GitHub: [fundmycause](https://github.com/eric-an/fundmycause){:target="_blank"} 

