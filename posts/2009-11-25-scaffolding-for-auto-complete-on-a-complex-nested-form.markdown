title: Scaffolding for auto complete on a complex, nested form
date: 2009/11/25
tag: View Mapper

<p>I just updated <a href="http://patshaughnessy.net/view_mapper">View Mapper</a> to work with <a href="http://patshaughnessy.net/repeated_auto_complete">my fork of the Rails auto_complete plugin</a> that allows for repeated text fields on the same complex form. This means that View Mapper can now generate scaffolding code that uses both nested attributes and the auto_complete plugin at the same time, to display a form like this:<br/><br/>
<img src="http://patshaughnessy.net/assets/2009/11/25/repeated_auto_complete.png"/> 
<p>To generate this sort of complex form for two of your models you&rsquo;ll first need to install my &ldquo;repeated_auto_complete&rdquo; gem from <a href="http://gemcutter.org/gems/repeated_auto_complete">gemcutter.org</a>:</p>
<pre>$ gem sources -a http://gemcutter.org
http://gemcutter.org added to sources
$ sudo gem install repeated_auto_complete
Successfully installed repeated_auto_complete-0.1.0
1 gem installed
Installing ri documentation for repeated_auto_complete-0.1.0...
Installing RDoc documentation for repeated_auto_complete-0.1.0...</pre>
<p>To learn more about repeated_auto_complete and what it does, see: <a href="http://patshaughnessy.net/repeated_auto_complete">http://patshaughnessy.net/repeated_auto_complete</a>. Now you can generate a complex form like the one shown above for two of your models in a has_many/belongs_to, has_and_belongs_to_many or has_many, :through association by installing View Mapper (version 0.3.1 or later):</p>
<pre>$ sudo gem install view_mapper
Successfully installed view_mapper-0.3.1
1 gem installed
Installing ri documentation for view_mapper-0.3.1...
Installing RDoc documentation for view_mapper-0.3.1...</pre>
<p>&hellip; and then running the &ldquo;view_for&rdquo; generator with a view option called &ldquo;has_many_auto_complete,&rdquo; like this:</p>
<pre>./script/generate view_for group --view has_many_auto_complete:people</pre>
<p>&nbsp;</p>
<p><b>Detailed Example</b></p>
<p>To see how easy it is to create a complex form using View Mapper, let&rsquo;s create one from scratch in a brand new Rails app. You should be able to follow along using the commands below on your machine. First, let&rsquo;s create a new Rails application:</p>
<pre>$ rails complex_auto_complete
      create  
      create  app/controllers
      create  app/helpers
      create  app/models
      create  app/views/layouts
      create  config/environments
      create  config/initializers
      create  config/locales
    &hellip; etc..
      create  log/server.log
      create  log/production.log
      create  log/development.log
      create  log/test.log</pre>
<p>The first thing I&rsquo;ll do is install the auto_complete plugin. However, since I&rsquo;m planning to use auto_complete on a complex form, I&rsquo;ll need to get my fork of auto_complete which I&rsquo;ve deployed as a gem on gemcutter.org:</p>
<pre>$ gem sources -a http://gemcutter.org
http://gemcutter.org added to sources
$ sudo gem install repeated_auto_complete
Successfully installed repeated_auto_complete-0.1.0
1 gem installed
Installing ri documentation for repeated_auto_complete-0.1.0...
Installing RDoc documentation for repeated_auto_complete-0.1.0...</pre>
<p>And let&rsquo;s update my new app to use the repeated_auto_complete gem by editing the config/environment.rb file:</p>
<pre>Rails::Initializer.run do |config|
&hellip;etc&hellip;
  <b>config.gem &quot;repeated_auto_complete&quot;</b>
&hellip;etc&hellip;</pre>
<p>If you prefer, you can also install this the old fashioned way, using &ldquo;script/plugin install git://github.com/patshaughnessy/auto_complete.git&rdquo;. Next, let&rsquo;s generate a new model called &ldquo;person&rdquo; with a couple of fields for name and age, like the ones shown above in the screen shot:</p>
<pre>$ cd complex_auto_complete/
$ ./script/generate model person name:string age:integer group_id:integer
      exists  app/models/
      exists  test/unit/
      exists  test/fixtures/
      create  app/models/person.rb
      create  test/unit/person_test.rb
      create  test/fixtures/people.yml
      create  db/migrate
      create  db/migrate/20091125195040_create_people.rb</pre>
<p>Note that I&rsquo;ve also included an integer field for the group id, since in a minute I&rsquo;ll be adding a belongs_to association for people to groups.</p>
<p>Now I&rsquo;m ready to use View Mapper&hellip; if you haven&rsquo;t installed that yet, get it from gemcutter.org like this:</p>
<pre>$ sudo gem install view_mapper
Successfully installed view_mapper-0.3.1
1 gem installed
Installing ri documentation for view_mapper-0.3.1...
Installing RDoc documentation for view_mapper-0.3.1...</pre>
<p>You&rsquo;ll need at least version 0.3.1 to use auto_complete on a complex form. Now I can use View Mapper to create scaffolding for a new &ldquo;group&rdquo; model that has many people with auto_complete like this:</p>
<pre>$ ./script/generate scaffold_for_view group name:string
                    --view has_many_auto_complete:people
       error  Table for model &#x27;person&#x27; does not exist
              - run rake db:migrate first.</pre>
<p>Yes&hellip; I forgot to create the people table in my database; if we do that:</p>
<pre>$ rake db:migrate
(in /Users/pat/rails-apps/complex_auto_complete)
==  CreatePeople: migrating ===================================================
-- create_table(:people)
   -&gt; 0.0014s
==  CreatePeople: migrated (0.0015s) ==========================================</pre>
<p>&hellip; and then re-run View Mapper:</p>
<pre>$ ./script/generate scaffold_for_view group name:string
                    --view has_many_auto_complete:people
     warning  Model Person does not contain a belongs_to
              association for Group.</pre>
<p>&hellip; we get a second error message! This time View Mapper is reminding me that I still need to add &ldquo;belongs_to :group&rdquo; to the person model in order to get the complex form to work. Let&rsquo;s do that now:</p>
<pre>class Person &lt; ActiveRecord::Base
  <b>belongs_to :group</b>
end</pre>
<p>And now I can run View Mapper once more:</p>
<pre>$ ./script/generate scaffold_for_view group name:string
                    --view has_many_auto_complete:people
      exists  app/models/
&hellip;etc&hellip;
      create  app/models/group.rb
      create  test/unit/group_test.rb
      create  test/fixtures/groups.yml
      exists  db/migrate
      create  db/migrate/20091125195715_create_groups.rb
      create  app/views/groups/show.html.erb
      create  app/views/groups/_form.html.erb
      create  app/views/groups/_person.html.erb
      create  public/javascripts/nested_attributes.js
       route  map.connect &#x27;auto_complete_for_group_name&#x27;,
                          :controller =&gt; &#x27;groups&#x27;,
                          :action =&gt; &#x27;auto_complete_for_group_name&#x27;
       route  map.connect &#x27;auto_complete_for_person_name&#x27;,
                          :controller =&gt; &#x27;groups&#x27;,
                          :action =&gt; &#x27;auto_complete_for_person_name&#x27;
       route  map.connect &#x27;auto_complete_for_person_age&#x27;,
                          :controller =&gt; &#x27;groups&#x27;,
                          :action =&gt; &#x27;auto_complete_for_person_age&#x27;</pre>
<p>Now you can see the new scaffolding files View Mapper created, including some new scaffolding files peculiar to complex forms, like &ldquo;nested_attributes.js,&rdquo; &ldquo;_form.html.erb,&rdquo; and &ldquo;_person.html.erb.&rdquo; You may also have noticed View Mapper added three new routes related to the auto_complete plugin; these will handle the AJAX requests used to return the auto_complete options to the form.</p>
<p>Now to get it all to work, I just need to create the group table:</p>
<pre>$ rake db:migrate
(in /Users/pat/rails-apps/complex_auto_complete)
==  CreateGroups: migrating ===================================================
-- create_table(:groups)
   -&gt; 0.0013s
==  CreateGroups: migrated (0.0014s) ==========================================</pre>
<p><br/>Now running my server and creating a new group I see:<br/><br/>
<img src="http://patshaughnessy.net/assets/2009/11/25/auto_complete_new_group.png"/> 
<p>If you click &ldquo;Add a Person&rdquo; you&rsquo;ll see nested fields for new Person records appear. This all works exactly the same way as the standard nested attributes scaffolding that I described in <a href="http://patshaughnessy.net/2009/11/9/scaffolding-for-complex-forms-using-nested-attributes">my last post</a>. The only difference is that in this form, each of the text fields present in both the parent (&ldquo;Group&rdquo;) and child (&ldquo;Person&rdquo;) models are displayed using the &ldquo;text_field_with_auto_complete&rdquo; method.</p>
<p>I&rsquo;ll try to write up a detailed walk through of how this scaffolding actually works as soon as I can&hellip; there are a lot of interesting details in the code that will be fun to look at. In the meantime, hopefully this scaffolding will make it easier for you to learn how to use auto_complete and nested attributes together in your app.</p>
