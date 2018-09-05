title: "ActiveRecord with large result sets - part 1: select_all vs. find"
date: 2010/09/04
tag: Ruby

<p>It&rsquo;s well documented that using ActiveRecord::Base.connection.select_all can speed up ActiveRecord database queries when you are expecting a large result set. This is because select_all just returns an array of hashes containing the results, and ActiveRecord doesn&rsquo;t have to do the work of instantiating a new model object for each row in the result.</p>
<p>Here&rsquo;s an example:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="co">ActiveRecord</span>::<span class="co">Base</span>.connection.select_all(<span class="s"><span class="dl">'</span><span class="k">SELECT * FROM users</span><span class="dl">'</span></span>)</pre></div>
</div><br>
<p>This returns an array of hashes, instead of user model objects:</p>
<div class="CodeRay">
  <div class="code"><pre>=&gt; [{&quot;name&quot;=&gt;&quot;pat&quot;,
     &quot;created_at&quot;=&gt;&quot;2010-09-03 16:59:24.097209&quot;,
     &quot;updated_at&quot;=&gt;&quot;2010-09-03 16:59:24.097209&quot;,
     &quot;id&quot;=&gt;1,
     &quot;email&quot;=&gt;&quot;pat@patshaughnessy.net&quot;}, etc...]</pre></div>
</div><br>
<p>You can also use &ldquo;select_values&rdquo; if you need just one value from a single database column:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="co">ActiveRecord</span>::<span class="co">Base</span>.connection.select_values(<span class="s"><span class="dl">'</span><span class="k">SELECT name FROM users</span><span class="dl">'</span></span>)

=&gt; [<span class="s"><span class="dl">&quot;</span>pat<span class="dl">&quot;</span></span>, etc... ]</pre></div>
</div><br>
<p>If select_all is faster than using a normal call to ActiveRecord::Base.find, why not use select_all for everything? The reason is that the extra speed select_all offers comes with a heavy price:
<ul>
  <li>You have to write the SQL statement yourself. Normally ActiveRecord generates the SQL statement for you automatically based on your associations, named scopes, etc.</li>
  <li>You have to use a hash to access the result, instead of a model object. In this example, you would have to write user[&ldquo;name&rdquo;] instead of user.name. Aside from being harder to read and type, with a hash you also lose the opportunity to call model methods to process the value somehow.</li></ul></p>
<p>So using select_all is a tradeoff: more speed vs. less coding convenience. Is it worth it?</p>
<p>Well, it might be if:
<ul>
  <li>You expect a large number of results from a database query - this will make the speed benefit more noticeable.</li>
  <li>It&rsquo;s easy to write the SQL select statement, because you don&rsquo;t have many different SQL statements to write based on different combinations of associations and scopes.</li>
  <li>Or if you need to use an unusual, hand crafted SQL statement that ActiveRecord wouldn&rsquo;t normally generate you might have to use select_all regardless of the speed benefit.</li>
</ul></p>
<p>This week I ran some Rails performance tests to measure the actual speed difference for a very simple find(:all) query with no associations. What I found was:
<ul>
  <li>For Rails 3, select_all was almost 2x times faster: it took ActiveRecord about 40% less time to return an array of hashes with select_all than it did to return an array of model objects.</li>
  <li>For Rails 2.3.8, there was a less noticeable difference: only a 22% improvement.</li>
  <li>Using Ruby 1.8.7 vs. Ruby 1.8.6 was much faster overall: about 2x faster for both select_all and find.</li>
  </ul></p>
<p>This rest of this article will show you how I setup and ran the performance tests, and what my detailed results were. Most of what I&rsquo;m going to do below is taken form an excellent Rails guide article by Pratik Naik: <a href="http://guides.rubyonrails.org/performance_testing.html">Performance Testing Rails Applications.</a></p>
<p><b>Setting up for a Rails performance test</b></p>
<p>I&rsquo;ll get started by creating a new Rails 3 app - using the 3.0.0 version which was released just last week!</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails new perf_test
      <span class="s">create</span>
      <span class="s">create  README</span>
      <span class="s">create  Rakefile</span>
      <span class="s">create  config.ru</span>
      <span class="s">create  .gitignore</span>
      <span class="s">create  Gemfile</span>
etc....
</pre></div>
</div><br>
<p>And now I&rsquo;ll create a user model from the example above:</p>
<div class="CodeRay">
  <div class="code"><pre>$ cd perf_test
$ rails generate model user name:string email:string
      invoke  active_record
      <span class="s">create</span>    db/migrate/20100903165238_create_users.rb
      <span class="s">create</span>    app/models/user.rb
      invoke    test_unit
      <span class="s">create</span>      test/unit/user_test.rb
      <span class="s">create</span>      test/fixtures/users.yml

$ rake db:migrate
(in /Users/pat/rails-apps/perf_test)
==  CreateUsers: migrating ==============================
-- create_table(:users)
   -&gt; 0.0013s
==  CreateUsers: migrated (0.0014s) =====================
</pre></div>
</div><br>
<p>Next I&rsquo;ll write a simple rake task in a file called lib/tasks/users.rake to create a specified number of users, so we can time queries based on different numbers of records. Also, having a separate rake task for this avoids the possibility of including the record create time in the performance tests.</p>
<div class="CodeRay">
  <div class="code"><pre>task <span class="sy">:populate_users</span> =&gt; <span class="sy">:environment</span> <span class="r">do</span>
  count = <span class="co">ENV</span>[<span class="s"><span class="dl">'</span><span class="k">count</span><span class="dl">'</span></span>].to_i
  <span class="co">User</span>.delete_all
  count.times <span class="r">do</span>
    <span class="co">User</span>.create <span class="sy">:name</span>  =&gt; <span class="s"><span class="dl">'</span><span class="k">pat</span><span class="dl">'</span></span>,
                <span class="sy">:email</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">pat@patshaughnessy.net</span><span class="dl">'</span></span>
  <span class="r">end</span>
  puts <span class="s"><span class="dl">&quot;</span><span class="k">User count: </span><span class="il"><span class="idl">#{</span><span class="co">User.count</span><span class="idl">}</span></span><span class="dl">&quot;</span></span>
<span class="r">end</span></pre></div>
</div><br>
<p>Now I&rsquo;ll create a new performance test using the Rails generator, and delete the test that came with the new app:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails generate performance_test load_users
      invoke  test_unit
      <span class="s">create</span>    test/performance/load_users_test.rb
$ rm test/performance/browsing_test.rb</pre></div>
</div><br>
<p>Editing the load_users_test.rb file, I&rsquo;ll add a couple simple tests that load all of the user records:</p>
<div class="CodeRay">
  <div class="code"><pre>require <span class="s"><span class="dl">'</span><span class="k">test_helper</span><span class="dl">'</span></span>
require <span class="s"><span class="dl">'</span><span class="k">rails/performance_test_help</span><span class="dl">'</span></span>

<span class="r">class</span> <span class="cl">LoadUsersTest</span> &lt; <span class="co">ActionDispatch</span>::<span class="co">PerformanceTest</span>
  <span class="r">def</span> <span class="fu">test_find</span>
    user_models = <span class="co">User</span>.find <span class="sy">:all</span>
    puts <span class="s"><span class="dl">&quot;</span><span class="k">Loaded </span><span class="il"><span class="idl">#{</span><span class="co">user_models.size</span><span class="idl">}</span></span><span class="k"> users</span><span class="dl">&quot;</span></span>
  <span class="r">end</span>

  <span class="r">def</span> <span class="fu">test_select_all</span>
    user_hashes =
      <span class="co">ActiveRecord</span>::<span class="co">Base</span>.connection.select_all(<span class="s"><span class="dl">'</span><span class="k">SELECT users.* FROM users</span><span class="dl">'</span></span>)
    puts <span class="s"><span class="dl">&quot;</span><span class="k">Loaded </span><span class="il"><span class="idl">#{</span><span class="co">user_hashes.size</span><span class="idl">}</span></span><span class="k"> users</span><span class="dl">&quot;</span></span>
  <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br>
<p>Finally, I&rsquo;ll install &ldquo;ruby-prof,&rdquo; a helpful profiling tool that will allow us to use the rake test:profile command:</p>
<div class="CodeRay">
  <div class="code"><pre>$ gem install ruby-prof
Building native extensions.  This could take a while...
Successfully installed ruby-prof-0.9.2
1 gem installed
Installing ri documentation for ruby-prof-0.9.2...
Installing RDoc documentation for ruby-prof-0.9.2...</pre></div>
</div><br>
<p>And I need to add this gem in my Gemfile:</p>
<div class="CodeRay">
  <div class="code"><pre>source <span class="s"><span class="dl">'</span><span class="k">http://rubygems.org</span><span class="dl">'</span></span>

gem <span class="s"><span class="dl">'</span><span class="k">rails</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">3.0.0</span><span class="dl">'</span></span>

<div class='container'>gem <span class="s"><span class="dl">'</span><span class="k">ruby-prof</span><span class="dl">'</span></span><span class='overlay'></span></div>

<span class="c"># Bundle edge Rails instead:</span>
<span class="c"># gem 'rails', :git =&gt; 'git://github.com/rails/rails.git'</span></pre></div>
</div><br>
<p>Ok - great! Now I&rsquo;ll create 10,000 users and then run the new performance tests:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rake populate_users count=10000
(in /Users/pat/rails-apps/perf_test)
User count: 10000

$ rake test:profile
(in /Users/pat/rails-apps/perf_test)
Loaded suite /Users/pat/.rvm/gems/...
Started
Loaded 2 users
LoadUsersTest#test_find (10 ms warmup)
Loaded 2 users
        process_time: 4 ms
              memory: unsupported
             objects: unsupported
.Loaded 2 users
LoadUsersTest#test_select_all (0 ms warmup)
Loaded 2 users
        process_time: 2 ms
              memory: unsupported
             objects: unsupported
.
Finished in 0.319634 seconds.

6 tests, 0 assertions, 0 failures, 0 errors</pre></div>
</div><br>
<p>Huh? What happened? Why were there only two users loaded? Well it turns out that the 2 users came from the test/fixtures/users.yml fixtures file create by the model generator I ran earlier:</p>
<div class="CodeRay">
  <div class="code"><pre>one:
  name: <span class="co">MyString</span>
  email: <span class="co">MyString</span>

two:
  name: <span class="co">MyString</span>
  email: <span class="co">MyString</span></pre></div>
</div><br>
<p>And the ten thousand users I created with my rake task were put into my development database... not my test database.</p>
<p><b>Retaining the contents of the test database between test suite runs</b></p>
<p>I can easily run my rake task using RAILS_ENV=test to fill the test database with users instead of the development database, but they will still be cleared out when the test database is purged and reloaded each time I run my profile test suite. What I really want to do is retain the contents of my test database each time I run the tests. Using code from a <a href="http://stackoverflow.com/questions/1097845/how-to-prevent-rake-test-to-call-task-dbtestprepare">helpful stack overflow discussion</a> on how to do this, I put this function into my Rakefile:</p>
<div class="CodeRay">
  <div class="code"><pre>require <span class="co">File</span>.expand_path(<span class="s"><span class="dl">'</span><span class="k">../config/application</span><span class="dl">'</span></span>, <span class="pc">__FILE__</span>)
require <span class="s"><span class="dl">'</span><span class="k">rake</span><span class="dl">'</span></span>

<div class='container'><span class="co">Rake</span>::<span class="co">TaskManager</span>.class_eval <span class="r">do</span>
  <span class="r">def</span> <span class="fu">remove_task</span>(task_name)
    <span class="iv">@tasks</span>.delete(task_name.to_s)
  <span class="r">end</span>
<span class="r">end</span><span class='overlay'></span></div>

<span class="co">PerfTest</span>::<span class="co">Application</span>.load_tasks</pre></div>
</div><br>
<p>...and I created a NOP task in my users.rake file, after calling remove_task:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="co">Rake</span>.application.remove_task <span class="s"><span class="dl">'</span><span class="k">db:test:prepare</span><span class="dl">'</span></span>

namespace <span class="sy">:db</span> <span class="r">do</span>
<span class="er"> </span> namespace <span class="sy">:test</span> <span class="r">do</span> 
<span class="er"> </span> <span class="er"> </span> task <span class="sy">:prepare</span> <span class="r">do</span> |t|
<span class="er"> </span> <span class="er"> </span> <span class="r">end</span>
<span class="er"> </span> <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br>
<p>Finally I need to be sure to delete the users.yml fixture file, or Rails will still delete and reload all of the users between each individual test:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rm test/fixtures/users.yml</pre></div>
</div><br>
<p><b>Just how much faster is select_all vs. find?</b></p>
<p>Ok - now I&rsquo;m all set to run some tests; let&rsquo;s start with 1000 users records:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rake populate_users count=1000 RAILS_ENV=test
(in /Users/pat/rails-apps/perf_test)
User count: 1000

$ rake test:profile
(in /Users/pat/rails-apps/perf_test)
Loaded suite /Users/pat/.rvm/gems/ruby-1.8.7...
Started
Loaded 1000 users
LoadUsersTest#test_find (54 ms warmup)
Loaded 1000 users
        process_time: 398 ms
              memory: unsupported
             objects: unsupported
.Loaded 1000 users
LoadUsersTest#test_select_all (33 ms warmup)
Loaded 1000 users
        process_time: 241 ms
              memory: unsupported
             objects: unsupported
.
Finished in 0.99688 seconds.

6 tests, 0 assertions, 0 failures, 0 errors</pre></div>
</div><br>
<p>Each performance profile test gives us three results: process time, memory usage and the number of Ruby objects created. However, since I&rsquo;m not using the &ldquo;GC Patched&rdquo; (garbage collection patch) version of Ruby I only get the process time value. In a future blog post I&rsquo;ll show how to update your Ruby interpreter with the patch that counts the number of objects created, and measures the amount of memory used. &ldquo;Process time&rdquo; refers to the amount of time used by the Ruby process, not the actual time elapsed (the &ldquo;wall time&rdquo;).</p>
<p>But for now, we can see that the first test, which uses ActiveRecord::Base.find, took 398ms to load 1000 rows from the SQLite database, and to return an array of 1000 user model objects. The second test, using ActiveRecord::Base.connection.select_all, took 241ms to load the same data but return it in the form of an array of hashes.</p>
<p>Let&rsquo;s increase the number of users to 10,000 and repeat the test:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rake populate_users count=10000 RAILS_ENV=test
(in /Users/pat/rails-apps/perf_test)
User count: 10000

$ rake test:profile
(in /Users/pat/rails-apps/perf_test)
Loaded suite /Users/pat/.rvm/gems/ruby-1.8.7...
Started
Loaded 10000 users
LoadUsersTest#test_find (476 ms warmup)
Loaded 10000 users
        process_time: 3.99 sec
              memory: unsupported
             objects: unsupported
.Loaded 10000 users
LoadUsersTest#test_select_all (378 ms warmup)
Loaded 10000 users
        process_time: 2.42 sec
              memory: unsupported
             objects: unsupported
.
Finished in 7.730976 seconds.

6 tests, 0 assertions, 0 failures, 0 errors</pre></div>
</div><br>
<p>It looks like the time taken for 10,000 users is simply 10x the amount taken for 1,000. In other words, this is a fairly linear process: it takes my laptop a certain number of milliseconds to process each user row: about 0.4 ms per row for ActiveRecord.find and 0.24ms for select_all.</p>
<p><b>Results</b></p>
<p>Here are my timings - I&rsquo;m using a MacBook Pro with a 2.6 GHz Intel Core 2 Duo processor. My database server is SQLite 3.7.0, via the sqlite3-ruby-1.3.1 gem. FYI the SQLite gem version seems to be important; using an older version of this gem slowed down the results dramatically.</p>
<p>
  Rails 3.0.0/Ruby 1.8.7:
</p>
<p>
  <table cellpadding="14" cellspacing="0" border="1" style="font-family:verdana">
    <tr align="center">
      <td  width="100"><b>records</b></td>
      <td  width="100"><b>select_all</b></td>
      <td  width="100"><b>find</b></td>
      <td  width="100"><b>delta</b></td>
    </tr>
    <tr align="right">
      <td  width="100">1000</td>
      <td  width="100">241ms</td>
      <td  width="100">398ms</td>
      <td  width="100">39%</td>
    </tr>
    <tr align="right">
      <td  width="100">10000</td>
      <td  width="100">2,420ms</td>
      <td  width="100">3,990ms</td>
      <td  width="100">39%</td>
    </tr>
    <tr align="right">
      <td  width="100">100000</td>
      <td  width="100">25,480ms</td>
      <td  width="100">42,580ms</td>
      <td  width="100">40%</td>
    </tr>
  </table>
  <br>
</p>
<p>
  Rails 2.3.8/Ruby 1.8.7:
</p>
<p>
  <table cellpadding="14" cellspacing="0" border="1" style="font-family:verdana">
    <tr align="center">
      <td  width="100"><b>records</b></td>
      <td  width="100"><b>select_all</b></td>
      <td  width="100"><b>find</b></td>
      <td  width="100"><b>delta</b></td>
    </tr>
    <tr align="right">
      <td  width="100">1000</td>
      <td  width="100">262ms</td>
      <td  width="100">336ms</td>
      <td  width="100">22%</td>
    </tr>
    <tr align="right">
      <td  width="100">10000</td>
      <td  width="100">2,660ms</td>
      <td  width="100">3,390ms</td>
      <td  width="100">22%</td>
    </tr>
    <tr align="right">
      <td  width="100">100000</td>
      <td  width="100">27,770ms</td>
      <td  width="100">35,490ms</td>
      <td  width="100">22%</td>
    </tr>
  </table>
  <br >
</p>
<p>
  Rails 2.3.5/Ruby 1.8.6:
</p>
<p>
  <table cellpadding="14" cellspacing="0" border="1" style="font-family:verdana">
    <tr align="center">
      <td  width="100"><b>records</b></td>
      <td  width="100"><b>select_all</b></td>
      <td  width="100"><b>find</b></td>
      <td  width="100"><b>delta</b></td>
    </tr>
    <tr align="right">
      <td  width="100">1000</td>
      <td  width="100">530ms</td>
      <td  width="100">651ms</td>
      <td  width="100">19%</td>
    </tr>
    <tr align="right">
      <td  width="100">10000</td>
      <td  width="100">5,350ms</td>
      <td  width="100">6,540ms</td>
      <td  width="100">18%</td>
    </tr>
    <tr align="right">
      <td  width="100">100000</td>
      <td  width="100">53,820ms</td>
      <td  width="100">67,030ms</td>
      <td  width="100">20%</td>
    </tr>
  </table>
</p>
