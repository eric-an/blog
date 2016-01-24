---
layout: post
title:  "db:rollback vs. db:migrate:down"
date:   2015-11-14
permalink: /rollback_migrations
---

It happens every once in a while - the database table is set up incorrectly in the migration file and you don't realize it until later. Or maybe you simply need to edit the database table based on updated information. Read on:

**Problem:**

<img src="{{ site.baseurl }}/img/migration_error.png" alt="migration_error" width="646" height="280" /><br>
*`:rating` is incorrectly identified in the comments database table as a string when it should be an integer*

**Solution A: db:rollback**
---

To rollback the latest migration file, you can run:

`$ rake db:rollback`

However, this will only roll back **the latest** migration file. In our example, two more migration files were run after `20151012175156_create_comments.rb` (the target migration file). Instead, use this command:

`$ rake db:rollback STEP=3`

`STEP=3` rolls back the database 3 migration files to the target migration file that we desire. <span style="text-decoration: underline;">However, this will rollback the migrations that happened after the target migration as well.</span>  This can be good or bad - it is useful if you ran migration files after the target migration that you no longer need and want to undo. In this case, we won't touch those other migration files.

**Solution B: db:migrate:down version=**
---

This is probably the better method to edit a target migration file, as it only rolls back the target migration file. Since we want to edit 20151012175156_create_comments.rb, run:

`$ rake db:migrate:down version=20151012175156`

where version is equal to the numbers (composed of date and time) located at the beginning of your migration file.

Whichever solution you choose, edit the migration file by changing "t.string" to "t.integer" and then run:

`$ rake db:migrate`

to update your database and finish. (In this example, Solution A will run the two migration files after the target migration file as well, while Solution B will only run the target migration file.)

**Alternative Solution: run a new migration file**
---

Instead of the potential anxiety that comes with having to edit history, you can run a new migration file to update the type cast of the :rating field in the comments table by dropping and re-adding the comments table. Run:

`$ rails generate migration UpdateTypeCastOfRatings`

This will create an empty migration file in which you can cleanly drop the old table from the database and add it again by copying and pasting the contents of the original target migration file to this new migration file. Your new migration file should look like this:

{% highlight ruby linenos %}
class UpdateTypeCastOfRatings < ActiveRecord::Migration
  def change
    drop_table :comments
    create_table :comments do |t|
      t.references :user, index: true, foreign_key: true
      t.text :body
      t.string :rating
      t.string :integer
      t.references :product, index: true, foreign_key: true

      t.timestamps null: false
    end
  end
end
{% endhighlight %}

Make sure to run  `$ rake db:migrate`  to update your database.

If you don't want to drop the entire table and create it again, you can use the `change_column` migration method on `:rating`. If interested, <a href="http://edgeguides.rubyonrails.org/active_record_migrations.html#changing-columns" target="_blank">read more about this method</a>.