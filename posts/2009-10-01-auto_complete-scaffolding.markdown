title: Auto_complete scaffolding
date: 2009/10/01
tag: View Mapper

<p>I&rsquo;ve written a lot here about the Rails auto_complete plugin; I&rsquo;ve also <a href="http://patshaughnessy.net/repeated_auto_complete">refactored the auto_complete plugin to support repeated fields and named scopes</a>. Today I&rsquo;d like to show how you can automatically generate Rails view and controller code with auto_complete behavior for one of your models using a new gem I&rsquo;ve written called <a href="http://patshaughnessy.net/view_mapper">View Mapper</a>. If you&rsquo;ve never used the auto_complete plugin before this is a great way to learn quickly how to use it in your app; even if you are familiar with the plugin using scaffolding like this can help to get a working auto_complete form up and running quickly and let you concentrate on more important parts of your app.</p>
<p>Let&rsquo;s say you have an existing model in your app called &ldquo;Person:&rdquo;</p>
<pre>Class Person &lt; ActiveRecord::Base
end</pre>
<p>And suppose the Person model has two string attributes for the person&rsquo;s name and the name of the office they work in:</p>
<pre>class CreatePeople &lt; ActiveRecord::Migration
  def self.up
    create_table :people do |t|
      t.string :name
      t.string :office
      etc&hellip;</pre>
<p>Now let&rsquo;s install View Mapper so we can generate an auto_complete view for our person model. Since I&rsquo;ve only deployed <a href="http://gemcutter.org/gems/view_mapper">view_mapper on gemcutter.org</a> for now, you&rsquo;ll also need to add gemcutter as a gem source if you haven&rsquo;t already.</p>
<pre>$ gem sources -a http://gemcutter.org
http://gemcutter.org added to sources
$ sudo gem install view_mapper
Successfully installed view_mapper-0.1.0
1 gem installed
Installing ri documentation for view_mapper-0.1.0...
Installing RDoc documentation for view_mapper-0.1.0...</pre>
<p>Now with view_mapper you can run a single command to generate scaffolding that displays the your existing person model&rsquo;s fields in a form with auto_complete type ahead behavior for the office field:</p>
<pre>$ ./script/generate view_for person --view auto_complete:office
      exists  app/controllers/
      exists  app/helpers/
      create  app/views/people
      exists  app/views/layouts/
      exists  test/functional/
      exists  test/unit/
      create  test/unit/helpers/
      exists  public/stylesheets/
      create  app/views/people/index.html.erb
      create  app/views/people/show.html.erb
      create  app/views/people/new.html.erb
      create  app/views/people/edit.html.erb
      create  app/views/layouts/people.html.erb
      create  public/stylesheets/scaffold.css
      create  app/controllers/people_controller.rb
      create  test/functional/people_controller_test.rb
      create  app/helpers/people_helper.rb
      create  test/unit/helpers/people_helper_test.rb
       route  map.resources :people
       route  map.connect &#x27;auto_complete_for_person_office&#x27;,
                          :controller =&gt; &#x27;people&#x27;,
                          :action =&gt; &#x27;auto_complete_for_person_office&#x27;</pre>
<p>This works just like the Rails scaffold generator, except that the view_for generator has also:</p>
<ul>
  <li>inspected your person model and found the name and office columns.</li>
  <li>added a route for &ldquo;auto_complete_for_person_office&rdquo; to routes.rb.</li>
  <li>added a call to &ldquo;auto_complete_for :person, :office&rdquo; to PersonController.</li>
  <li>used text_field_for_auto_complete on the office field in your new and edit forms.</li>
  <li>inserted &ldquo;javascript_include_tag :defaults&rdquo; into views/layouts/people.html.erb to load the prototype javascript library.</li>
</ul>
<p>If you start up your application and create a few person records with names and addresses, then you will see the auto_complete plugin working!</p>
<p><img src="http://patshaughnessy.net/assets/2009/10/1/person-autocomplete.png"/></p>
<p>With this working example right inside your application, you can easily review exactly how the view, route and controller files use auto_complete. After that you can adapt the view to fit into your application&rsquo;s design and delete the scaffolding you don&rsquo;t really need or want.</p>
<p>As another example, let&rsquo;s create an entirely new Rails application completely from scratch, and use View Mapper to setup auto_complete inside it:</p>
<pre>$ rails auto_complete_example
      create  
      create  app/controllers
      create  app/helpers
      create  app/models
      create  app/views/layouts
      etc&hellip;
$ cd auto_complete_example</pre>
<p>First, let&rsquo;s install the auto_complete plugin. (In a future post I&rsquo;ll show how to use View Mapper with my fork of auto_complete in a complex form.)</p>
<pre>$ ./script/plugin install git://github.com/rails/auto_complete.git
Initialized empty Git repository in /Users/pat/rails-apps/auto_complete_example/vendor/plugins/auto_complete/.git/
remote: Counting objects: 13, done.
remote: Compressing objects: 100% (12/12), done.
remote: Total 13 (delta 2), reused 0 (delta 0)
Unpacking objects: 100% (13/13), done.
From git://github.com/rails/auto_complete
 * branch            HEAD       -&gt; FETCH_HEAD</pre>
<p>Now that we have the plugin installed, let&rsquo;s create our scaffolding. Along with the 
 &ldquo;view_for&rdquo; generator I used above, View Mapper also provides a generator called &ldquo;scaffold_for_view.&rdquo; This works the same way, except it just creates a new model the same way the Rails scaffold generator does, instead of inspecting an existing model.</p>
<p>Let&rsquo;s create the same person model we used above, and an auto_complete view:</p>
<pre>$ ./script/generate scaffold_for_view person name:string office:string
                                      --view auto_complete:office
      exists  app/models/
      exists  app/controllers/
      exists  app/helpers/
      create  app/views/people
      exists  app/views/layouts/
      exists  test/functional/
      exists  test/unit/
      create  test/unit/helpers/
      exists  public/stylesheets/
      create  app/views/people/index.html.erb
      create  app/views/people/show.html.erb
      create  app/views/people/new.html.erb
      create  app/views/people/edit.html.erb
      create  app/views/layouts/people.html.erb
      create  public/stylesheets/scaffold.css
      create  app/controllers/people_controller.rb
      create  test/functional/people_controller_test.rb
      create  app/helpers/people_helper.rb
      create  test/unit/helpers/people_helper_test.rb
       route  map.resources :people
  dependency  model
      exists    app/models/
      exists    test/unit/
      exists    test/fixtures/
      create    app/models/person.rb
      create    test/unit/person_test.rb
      create    test/fixtures/people.yml
      create    db/migrate
      create    db/migrate/20091001161349_create_people.rb
       route  map.connect &#x27;auto_complete_for_person_office&#x27;,
                          :controller =&gt; &#x27;people&#x27;,
                          :action =&gt;&#x27;auto_complete_for_person_office&#x27;</pre>
<p>Note the syntax is the same as the standard Rails scaffold generator, except I&rsquo;ve added the &ldquo;view&rdquo; parameter to specify we want the auto_complete plugin to be used in the view.</p>
<p>Now we just need to migrate the database schema and run our app:</p>
<pre>$ rake db:migrate
(in /Users/pat/rails-apps/auto_complete_example)
==  CreatePeople: migrating ===================================================
-- create_table(:people)
   -&gt; 0.0012s
==  CreatePeople: migrated (0.0015s) ==========================================</pre>
<p>And if you add a few records, you&rsquo;ll see auto_complete working!</p>
<p>I&rsquo;ll be adding more views to View Mapper soon, and in future posts here I&rsquo;ll write about how to generate scaffolding for Paperclip file attachments, my version of auto_complete used on a complex form, and also how to write your own views for View Mapper.</p>
