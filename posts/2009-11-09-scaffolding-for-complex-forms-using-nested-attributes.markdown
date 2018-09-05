title: Scaffolding for complex forms using nested attributes
date: 2009/11/09
tag: View Mapper

<p>While the new nested attributes feature in Rails 2.3 greatly simplifies writing forms that operate on two or more models at the same time, writing a complex form is still a confusing and daunting task even for experienced Rails developers. To make this easier, I just added nested attribute support to my <a href="http://patshaughnessy.net/view_mapper">View Mapper gem</a>. This means you can generate complex form scaffolding for two or more models in a has_many/belongs_to, has_and_belongs_to_many or has_many, through relationship.</p>
<p><b>Example:</b></p>
<p>If I have a group model that has many people and accepts nested attributes for them like this:</p>
<pre>class Group &lt; ActiveRecord::Base
  has_many :people
  accepts_nested_attributes_for :people, :allow_destroy =&gt; true
end</pre>
<p>&hellip; and a person model that belongs to a group:</p>
<pre>class Person &lt; ActiveRecord::Base
  belongs_to :group
end</pre>
<p>&hellip; then View Mapper will allow you to generate scaffolding that displays groups of people all at once, like this:</p>
<pre>$ ./script/generate view_for group --view has_many:people
      exists  app/controllers/
      exists  app/helpers/
      create  app/views/groups
&hellip;etc&hellip;
      create  app/views/groups/_form.html.erb
      create  app/views/groups/_person.html.erb
      create  public/javascripts/nested_attributes.js</pre>
<p>Now if I open my Rails app and create a new group, I will see:<br/>
 <img src="http://patshaughnessy.net/assets/2009/11/9/new_group.png"/></p>
<p>This looks just like the standard Rails scaffolding, but with one additional &ldquo;Add a Person&rdquo; link. If you click on it, you&rsquo;ll see the attributes of the person model appear along with a &ldquo;remove&rdquo; link, indented to the right:<br/>
 <img src="http://patshaughnessy.net/assets/2009/11/9/new_group_detail.png"/></p>
<p>If I enter some values and submit, ActiveRecord will insert a new record into both the groups table and the people table, and set the group_id value in the new person record correctly:<br/>
 <img src="http://patshaughnessy.net/assets/2009/11/9/show_group.png"/><br/></p>
<p>View Mapper has:
<ul>
  <li>inspected your group and person models to find their attributes (columns).</li>
  <li>validated that they are in a has_many / belongs_to relationship, or in a has_and_belongs_to_many or a has_many, through relationship.</li>
  <li>checked that you have a foreign key column (&ldquo;group_id&rdquo; by default for this example) in the people table if necessary. (The foreign key isn&rsquo;t in the people table for has_and_belongs_to_many or has_many, through.)</li>
  <li>generated scaffolding using your attribute and model names, and that uses Javascript to support the &ldquo;Add a person&rdquo; and &ldquo;remove&rdquo; links.</li>
</ul></p>
<p>To get the add/remove links to work, I used a simplified version of the &ldquo;complex-form-examples&rdquo; sample application from <a href="http://github.com/ryanb/complex-form-examples">Ryan Bates</a> and <a href="http://github.com/alloy/complex-form-examples">Eloy Duran</a>. Ryan has <a href="http://railscasts.com/episodes/73-complex-forms-part-1">a few screen casts</a> on this topic as well. In my next post I&rsquo;ll explain how that works in detail, since understanding the details about how scaffolding works is the first step towards using it successfully in your app.</p>
<p>But for now, you can try this on your machine using the precise commands below&hellip;</p>
<p><b>Creating a new complex form from scratch</b></p>
<p>Let&rsquo;s get started by creating a new Rails application; you will need to have Rails 2.3 or later in order to make this work:</p>
<pre>$ rails complex-form
      create  
      create  app/controllers
      create  app/helpers
      create  app/models
      create  app/views/layouts
&hellip; etc &hellip;
      create  log/production.log
      create  log/development.log
      create  log/test.log</pre>
<p>Using the same group has many people example from above, let&rsquo;s generate a new person model:</p>
<pre>$ cd complex-form
$ ./script/generate model person first_name:string last_name:string
      exists  app/models/
      exists  test/unit/
      exists  test/fixtures/
      create  app/models/person.rb
      create  test/unit/person_test.rb
      create  test/fixtures/people.yml
      create  db/migrate
      create  db/migrate/20091109204744_create_people.rb</pre>
<p>And let&rsquo;s run that migration to create the people table:</p>
<pre>$ rake db:migrate
(in /Users/pat/rails-apps/complex-form)
==  CreatePeople: migrating ===================================================
-- create_table(:people)
   -&gt; 0.0013s
==  CreatePeople: migrated (0.0014s) ==========================================</pre>
<p>Now we&rsquo;re ready to run View Mapper. View Mapper contains two generators; one is for creating scaffolding for an existing model, called &ldquo;view_for,&rdquo; which is what I used above. There&rsquo;s also another generator called &ldquo;scaffold_for_view&rdquo; which will create a new model along with the scaffolding, using the same syntax as the standard Rails scaffold generator. Let&rsquo;s use that here, since we have a new app and haven&rsquo;t created the group model yet:</p>
<pre>$ ./script/generate scaffold_for_view group name:string --view has_many:people
     warning  Model Person does not contain a belongs_to association for Group.</pre>
<p>Here View Mapper is reminding me that I didn&rsquo;t specify &ldquo;belongs_to&rdquo; in the person model. This saves me the trouble later of figuring out what&rsquo;s wrong when my complex form doesn&rsquo;t work. Let&rsquo;s add that line to app/models/person.rb and try again:</p>
<pre>class Person &lt; ActiveRecord::Base
  <b>belongs_to :group</b>
end

$ ./script/generate scaffold_for_view group name:string --view has_many:people
     warning  Model Person does not contain a foreign key for Group.</pre>
<p>Duh&hellip; I also forgot to include the &ldquo;group_id&rdquo; column when I generated the person model. I could have done that by including &ldquo;group_id:integer&rdquo; on the script/generate model command line above. Since I already have the person model now, let&rsquo;s just continue by creating a new migration for the missing column:</p>
<pre>$ ./script/generate migration add_group_id_column_to_people
      exists  db/migrate
      create  db/migrate/20091109205711_add_group_id_column_to_people.rb</pre>
<p>Editing the migration file:</p>
<pre>class AddGroupIdColumnToPeople &lt; ActiveRecord::Migration
  def self.up
    <b>add_column :people, :group_id, :integer</b>
  end
etc&hellip;</pre>
<p>And running the migration:</p>
<pre>$ rake db:migrate
(in /Users/pat/rails-apps/complex-form)
==  AddGroupIdColumnToPeople: migrating =======================================
-- add_column(:people, :group_id, :integer)
   -&gt; 0.0010s
==  AddGroupIdColumnToPeople: migrated (0.0012s) ==============================</pre>
<p>Now let&rsquo;s run View Mapper once more to see whether we have any other problems, or whether we&rsquo;re ready to generate the complex form scaffolding:</p>
<pre>$ ./script/generate scaffold_for_view group name:string --view has_many:people
      exists  app/models/
      exists  app/controllers/
&hellip;etc&hellip;
      create  app/models/group.rb
      create  test/unit/group_test.rb
      create  test/fixtures/groups.yml
      exists  db/migrate
      create  db/migrate/20091109210312_create_groups.rb
      create  app/views/groups/show.html.erb
      create  app/views/groups/_form.html.erb
      create  app/views/groups/_person.html.erb
      create  public/javascripts/nested_attributes.js</pre>
<p>It worked! Just looking at the list of files that View Mapper created, you can get a sense of how it has customized the standard Rails scaffolding to implement the complex form: _form.html.erb, _person.html.erb, nested_attributes.js. More on these details in my next article.</p>
<p>One detail I will point out now is that in order to get you started in the right direction and to allow the complex form to work immediately, the scaffold_for_view generator included the has_many and accepts_nested_attributes_for calls in the new model:</p>
<pre>class Group &lt; ActiveRecord::Base
  has_many :people
  accepts_nested_attributes_for :people,
                                :allow_destroy =&gt; true,
                                :reject_if =&gt; proc { |attrs|
                                  attrs[&#x27;first_name&#x27;].blank? &amp;&amp;
                                  attrs[&#x27;last_name&#x27;].blank?
                                }
end</pre>
<p>You don&rsquo;t need to type in all of this code yourself and know the precise syntax of the accepts_nested_attributes_for method&hellip; it&rsquo;s all generated for you. Later when you start to customize the scaffolding to work for your specific requirements, you&rsquo;ll have a working example to look at right inside your app.</p>
<p>Finally, we&rsquo;re need to run the migrations once more since the scaffold_for_view generator created a new group model and corresponding migration for the groups table:</p>
<pre>$ rake db:migrate
(in /Users/pat/rails-apps/complex-form)
==  CreateGroups: migrating ===================================================
-- create_table(:groups)
   -&gt; 0.0013s
==  CreateGroups: migrated (0.0014s) ==========================================</pre>
<p>Now if you start up Rails and hit http://localhost:3000/groups/new, you&rsquo;ll see the complex form!</p>
