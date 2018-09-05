title: "ActiveRecord with large result sets - part 2: streaming data"
date: 2010/10/11
tag: Ruby

<p>In <a href="http://patshaughnessy.net/2010/9/4/activerecord-with-large-result-sets-part-1-select_all-vs-find">part 1 of this series</a> I showed how using select_all instead of find(:all) can often speed up slow ActiveRecord queries when you expect a large amount of data, by as much as 30 or 40% with Rails 3. However, often 30-40% faster just isn&rsquo;t enough. What difference does it make if I have to wait 3 minutes or 5 minutes for Rails to display a web page? Either way I&rsquo;m going to get frustrated and abandon the web site for something more interesting.</p>
<p>But let&rsquo;s suppose a Rails web site sets my expectations a little differently. As an example, let&rsquo;s say that I need to download a report containing the entire list of user accounts from a database table. When I click the &ldquo;download user report&rdquo; link, suppose I see this in Firefox:</p>
<p><img src="http://patshaughnessy.net/assets/2010/10/11/open-users.png"></p>
<p>Now instead of waiting to see a web page listing all of the users, I&rsquo;m waiting to download a large text or spreadsheet file containing the user list. Already, the web site has lowered my performance expectations, since I&rsquo;ve downloaded large files from the Internet many times before and I know it might take a few minutes to complete. Now when I click OK, Firefox displays this progress bar:</p>
<p><img src="http://patshaughnessy.net/assets/2010/10/11/progress-bar.png"></p>
<p>Looking at this progress bar I&rsquo;m more willing to wait. Besides displaying the animation, Firefox is also telling me how fast the data is being downloaded, and also how much I&rsquo;ve downloaded in total. Even though it tells me there an &ldquo;Unknown time remaining&rdquo; I&rsquo;m less likely to get impatient since I know data is being steadily downloaded. Finally, when the request is finished I can double click to open the file:</p>
<p><img src="http://patshaughnessy.net/assets/2010/10/11/finished-download.png"></p>
<p>The point here is that web site performance is often more about perception than reality. If it takes 5 minutes for Rails to display a web page no one will wait regardless of how important the data is to them, since in this case the web site seems broken and there&rsquo;s no indication anything is happening. But if it takes 5 minutes to download a large file many users will happily wait, since they know their data is steadily arriving from the server and that it&rsquo;s just a matter of time.</p>
<h2>How to implement this in your controller</h2>
<p>Using the user/name/email example from <a href="http://patshaughnessy.net/2010/9/4/activerecord-with-large-result-sets-part-1-select_all-vs-find">part 1</a>, here&rsquo;s the controller code you&rsquo;ll need to implement this in a Rails 2.x application:
<div class="CodeRay">
  <div class="code"><pre><span class="r">def</span> <span class="fu">index</span>
  respond_to <span class="r">do</span> |format|
    format.text <span class="r">do</span>
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-Type</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">&quot;</span><span class="k">text/plain</span><span class="dl">&quot;</span></span>
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-disposition</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">'</span><span class="k">attachment; filename=&quot;users.txt&quot;</span><span class="dl">'</span></span>
      render <span class="sy">:text</span> =&gt; <span class="r">proc</span> { |response, output|
        <span class="co">User</span>.find_each <span class="r">do</span> |u|
          output.write <span class="s"><span class="dl">&quot;</span><span class="il"><span class="idl">#{</span></span></span>u.name<span class="s"><span class="il"><span class="idl">}</span></span><span class="k"> </span><span class="il"><span class="idl">#{</span></span></span>u.email<span class="s"><span class="il"><span class="idl">}</span></span><span class="ch">\n</span><span class="dl">&quot;</span></span>
        <span class="r">end</span>
      }
    <span class="r">end</span>
  <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br></p>
<p>It turns out in Rails 3.0 render :text doesn&rsquo;t support passing a proc, so you&rsquo;ll need this code instead:

<div class="CodeRay"> 
  <div class="code"><pre><span class="r">def</span> <span class="fu">index</span> 
  respond_to <span class="r">do</span> |format|
    format.text <span class="r">do</span> 
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-Type</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">&quot;</span><span class="k">text/plain</span><span class="dl">&quot;</span></span> 
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-disposition</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">'</span><span class="k">attachment; filename=&quot;users.txt&quot;</span><span class="dl">'</span></span> 
<div class='container'>      erroneous_call_to_proc = <span class="pc">false</span> 
      <span class="pc">self</span>.response_body = proc { |response, output|
        <span class="r">unless</span> erroneous_call_to_proc
          <span class="co">User</span>.find_each <span class="r">do</span> |u|
            output.write <span class="s"><span class="dl">&quot;</span><span class="il"><span class="idl">#{</span>u.name<span class="idl">}</span></span><span class="k"> </span><span class="il"><span class="idl">#{</span>u.email<span class="idl">}</span></span><span class="ch">\n</span><span class="dl">&quot;</span></span> 
          <span class="r">end</span> 
        <span class="r">end</span> 
        erroneous_call_to_proc = <span class="pc">true</span> 
      }<span class='overlay'></span></div>    <span class="r">end</span> 
  <span class="r">end</span> 
<span class="r">end</span></pre></div> 
</div><br></p>
<p>In both Rails 2.x and Rails 3, the key here is that instead of using a Rails view to render my page, I provide a proc which is later called with the response object and the output stream. The proc can then repeatedly call output.write to stream the response to the client, one user at a time.</p>
<h2>Code walk through</h2>

<p>There are a few different things going on in this code snippet, so let&rsquo;s take them one at a time. First, I indicate to the client that the response is going to be a file, and not a standard web page by setting the &ldquo;Content-Type&rdquo; and &ldquo;Content-Disposition&rdquo; response headers like this:
<div class="CodeRay">
  <div class="code"><pre>headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-Type</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">&quot;</span><span class="k">text/plain</span><span class="dl">&quot;</span></span>
headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-disposition</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">'</span><span class="k">attachment; filename=&quot;users.txt&quot;</span><span class="dl">'</span></span></pre></div>
</div><br></p>
<p>Now the browser knows it&rsquo;s going to receive a text file called &ldquo;users.txt.&rdquo; Note that normally in a Rails app you would use send_file or send_data to set these headers, but these functions don&rsquo;t support streaming data in this manner. (Hmm... sounds like an opportunity for a Rails patch or plugin.) Also, if you knew ahead of time exactly how much data you were going to send to the client, you could also set the &ldquo;Content-Size&rdquo; header allowing the browser to display the total download size and correctly calculate the download time remaining.</p>
<p>Next, I call render :text and provide a proc instead of the actual text response. Rails will return from the controller action immediately, but later call the proc when it&rsquo;s time to generate the response.
<div class="CodeRay">
  <div class="code"><pre>render <span class="sy">:text</span> =&gt; proc { |response, output|</pre></div>
</div><br></p>
<p>As I said earlier, for Rails 3.0 you need to directly set self.response_body, instead of calling render :text.
<div class="CodeRay">
  <div class="code"><pre><span class="pc">self</span>.response_body = proc { |response, output|
</pre></div>
</div><br></p>
<p>Finally, inside the proc, I have two variables available to me: the response object and also the output stream. By calling output.write, I&rsquo;m able to send whatever text I would like to the client, which will be immediately streamed to the network. In this example, what I do inside the response proc is call User.find_each to iterate over all of the user records, calling output.write for each one.
<div class="CodeRay">
  <div class="code"><pre><span class="co">User</span>.find_each <span class="r">do</span> |u|
  output.write <span class="s"><span class="dl">&quot;</span><span class="il"><span class="idl">#{</span>u.name<span class="idl">}</span></span><span class="k"> </span><span class="il"><span class="idl">#{</span>u.email<span class="idl">}</span></span><span class="ch">\n</span><span class="dl">&quot;</span></span>
<span class="r">end</span></pre></div>
</div><br></p>
<p>Note: right now for Rails 3 it turns out <a href="https://rails.lighthouseapp.com/projects/8994-ruby-on-rails/tickets/4554-render-text-proc-regression">there’s a bug</a> that causes the response proc to be called one extra time. To avoid repeating all of the SQL queries unnecessarily, I’ve added a bit of code to ignore the second call to the proc:
<div class="CodeRay"> 
  <div class="code"><pre><span class="r">def</span> <span class="fu">index</span> 
  respond_to <span class="r">do</span> |format|
    format.text <span class="r">do</span> 
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-Type</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">&quot;</span><span class="k">text/plain</span><span class="dl">&quot;</span></span> 
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-disposition</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">'</span><span class="k">attachment; filename=&quot;users.txt&quot;</span><span class="dl">'</span></span> 
<div class='container'>      erroneous_call_to_proc = <span class="pc">false</span> <span class='overlay'></span></div>      <span class="pc">self</span>.response_body = proc { |response, output|
<div class='container'>        <span class="r">unless</span> erroneous_call_to_proc<span class='overlay'></span></div>          <span class="co">User</span>.find_each <span class="r">do</span> |u|
            output.write <span class="s"><span class="dl">&quot;</span><span class="il"><span class="idl">#{</span>u.name<span class="idl">}</span></span><span class="k"> </span><span class="il"><span class="idl">#{</span>u.email<span class="idl">}</span></span><span class="ch">\n</span><span class="dl">&quot;</span></span> 
          <span class="r">end</span> 
<div class='container'>        <span class="r">end</span> 
        erroneous_call_to_proc = <span class="pc">true</span> <span class='overlay'></span></div>      }
    <span class="r">end</span> 
  <span class="r">end</span> 
<span class="r">end</span></pre></div> 
</div> 
</p>
<p>Yes, this is super ugly but for now might be the only option for streaming in Rails 3. Hopefully in an upcoming version of Rails 3.0.x this will be fixed. I might even try to submit a Rails patch myself if I have time.</p>

<h2>find_each vs. find: memory usage</h2>
<p>Besides using a proc to stream the result, the other important detail here is the use of find_each instead of find :all. Calling find_each allows me to avoid loading the entire data set into memory all at once. If I have 10,000 or 100,000 records in the users table, not only do I have to worry about taking a long time to return all the records in a single HTTP request, but I also have to worry about Rails using up all of the available memory. Calling User.all or User.find :all would return an array 10,000 or 100,000 user model objects, all loaded into memory at one time. If there were many long columns in each user record, this might quickly use up all of the available memory on my server.</p>
<p>However, find_each iterates through the user records in batches, yielding each user record to my block one at a time. By default 1000 user records are loaded at a time, meaning that only 1000 user objects are loaded into memory at any given time, instead of 10,000 or 100,000. See <a href="http://ryandaigle.com/articles/2009/2/23/what-s-new-in-edge-rails-batched-find">Ryan Daigle&rsquo;s nice rundown</a> or the <a href="http://guides.rubyonrails.org/active_record_querying.html">Rails documentation</a> for more details on find_each.</p>
<p>If you take a look at your Rails log file, you&rsquo;ll see how find_each loads the user records in batches of 1000, using the primary key of the last record loaded as a filter for each subsequent query:
<div class="CodeRay">
  <div class="code"><pre>User Load (20.6ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt;= 0) ORDER BY users.id ASC LIMIT 1000
User Load (21.8ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 16632) ORDER BY users.id ASC LIMIT 1000
User Load (31.3ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 17632) ORDER BY users.id ASC LIMIT 1000
User Load (21.2ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 18632) ORDER BY users.id ASC LIMIT 1000
User Load (21.2ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 19632) ORDER BY users.id ASC LIMIT 1000
User Load (21.1ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 20632) ORDER BY users.id ASC LIMIT 1000
User Load (62.5ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 21632) ORDER BY users.id ASC LIMIT 1000
User Load (21.2ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 22632) ORDER BY users.id ASC LIMIT 1000
User Load (22.0ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 23632) ORDER BY users.id ASC LIMIT 1000
User Load (21.7ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 24632) ORDER BY users.id ASC LIMIT 1000
User Load (0.2ms)  SELECT &quot;users&quot;.* FROM &quot;users&quot; WHERE (&quot;users&quot;.&quot;id&quot; &gt; 25632) ORDER BY users.id ASC LIMIT 1000

etc...

</pre></div>
</div><br></p>
<p>Here the last record loaded by the first SQL query had an id of 16632, and find_each added 1000 to that id value each time to obtain the next 1000 users.</p>
<h2>Streaming data on your machine with Rails 3 and Mongrel</h2>
<p>The other caveat with the streaming code I show above is that it relies on the underlying  web server to stream the results properly via Rack to the user or client. Practically speaking, this means that you need to use Apache/Passenger, Mongrel or some other modern Rails stack to stream the response - but not Webrick. If you try the code above with Webrick you&rsquo;ll have to wait for the entire response to be loaded into memory and downloaded all at once before Firefox can display the file-open dialog box; in other words the data won&rsquo;t be streamed at all.</p>
<p>To wrap things up for today, let&rsquo;s build a sample Rails 3 app together right now from scratch that demonstrates file streaming:
<div class="CodeRay">
  <div class="code"><pre>$ rails -v
Rails 3.0.0
$ rails new stream_users
      <span class="s">create</span>  
      <span class="s">create</span>  README
      <span class="s">create</span>  Rakefile
      <span class="s">create</span>  config.ru
      <span class="s">create</span>  .gitignore
      <span class="s">create</span>  Gemfile
      <span class="s">create</span>  app
      <span class="s">create</span>  app/controllers/application_controller.rb
etc...</pre></div>
</div><br></p>
<div class="CodeRay">
  <div class="code"><pre>$ cd stream_users/
$ rails generate scaffold user name:string email:string
$ rake db:migrate</pre></div>
</div><br></p>
<p>And finally I&rsquo;ll create a large data set in the console... this takes a few minutes for my laptop to complete:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails console
Loading development environment (Rails 3.0.0)
ruby-1.8.7-p302 > 100000.times do
ruby-1.8.7-p302 >     User.create :name=>'pat', :email=>'pat@patshaughnessy.net'
ruby-1.8.7-p302 ?>  end
 => 10</pre></div>
 </div><br></p>
<p>Now let&rsquo;s take a look at the index action in the users_controller.rb file created by the scaffolding generator:
<div class="CodeRay">
  <div class="code"><pre>  <span class="r">def</span> <span class="fu">index</span>
    <span class="iv">@users</span> = <span class="co">User</span>.all

    respond_to <span class="r">do</span> |format|
      format.html <span class="c"># index.html.erb</span>
      format.xml  { render <span class="sy">:xml</span> =&gt; <span class="iv">@users</span> }
    <span class="r">end</span>
  <span class="r">end</span>
</pre></div>
</div><br></p>
<p>As I explained above, this will load all of the user records into memory at one time. By using a <a href="http://patshaughnessy.net/2010/9/28/ruby187gc-patch">GC patched version of the Ruby interpreter</a>, we can see how much memory this will take:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails console
Loading development environment (Rails 3.0.0)
ruby-1.8.7-p302 > GC.enable_stats
 => false 
ruby-1.8.7-p302 > users = User.all; nil
 => nil 
ruby-1.8.7-p302 > GC.allocated_size
 => 129630037 </pre></div>
 </div><br></p>
<p>That&rsquo;s 129MB of data loaded into memory just for this simple &ldquo;user&rdquo; example with only two string columns! And I still haven&rsquo;t tried to call my index.html.erb view file to render the web page.</p>
<p>Let&rsquo;s repeat the same test using find_each, but I&rsquo;ll call &ldquo;break&rdquo; as soon as my block is called for the first time:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails console
Loading development environment (Rails 3.0.0)
ruby-1.8.7-p302 > GC.enable_stats
 => false 
ruby-1.8.7-p302 > User.find_each do |u|
ruby-1.8.7-p302 >     break
ruby-1.8.7-p302 ?>  end
 => nil 
ruby-1.8.7-p302 > GC.allocated_size
 => 2186329 </pre></div>
 </div><br></p>
<p>Here I can see that only 2MB of memory is allocated. Of course as Rails iterates over all the user records more and more user objects will be created, but since the response proc streams the data out to the client each time the block is called, the Ruby garbage collector will free memory just as fast as it&rsquo;s allocated.</p>
<p>Returning to the controller, I&rsquo;ll paste in the streaming code from above:
<div class="CodeRay">
  <div class="code"><pre><span class="r">def</span> <span class="fu">index</span>
  respond_to <span class="r">do</span> |format|
    format.text <span class="r">do</span>
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-Type</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">&quot;</span><span class="k">text/plain</span><span class="dl">&quot;</span></span>
      headers[<span class="s"><span class="dl">&quot;</span><span class="k">Content-disposition</span><span class="dl">&quot;</span></span>] = <span class="s"><span class="dl">'</span><span class="k">attachment; filename=&quot;users.txt&quot;</span><span class="dl">'</span></span>
      <span class="pc">self</span>.response_body = proc { |response, output|
        <span class="co">User</span>.find_each <span class="r">do</span> |u|
          output.write <span class="s"><span class="dl">&quot;</span><span class="il"><span class="idl">#{</span>u.name<span class="idl">}</span></span><span class="k"> </span><span class="il"><span class="idl">#{</span>u.email<span class="idl">}</span></span><span class="ch">\n</span><span class="dl">&quot;</span></span>
        <span class="r">end</span>
      }
    <span class="r">end</span>
  <span class="r">end</span>
<span class="r">end</span>
</pre></div>
</div><br></p>
<p>Now let&rsquo;s try using our new app:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails server mongrel
Exiting
.../rack/handler/mongrel.rb:1:in `require':
    no such file to load -- mongrel (LoadError)
.../rack/handler/mongrel.rb:1
.../rack/handler.rb:17:in `const_get'
etc...</pre></div>
</div><br></p>
<p>Oops - it turns out that with Rails 3 you need to explicitly load the mongrel gem in your Gemfile:
<div class="CodeRay">
  <div class="code"><pre>source <span class="s"><span class="dl">'</span><span class="k">http://rubygems.org</span><span class="dl">'</span></span>

gem <span class="s"><span class="dl">'</span><span class="k">rails</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">3.0.0</span><span class="dl">'</span></span>

<div class='container'>gem <span class="s"><span class="dl">'</span><span class="k">mongrel</span><span class="dl">'</span></span><span class='overlay'></span></div>
<span class="c"># Bundle edge Rails instead:</span>

<span class="c"># gem 'rails', :git =&gt; 'git://github.com/rails/rails.git'</span>

gem <span class="s"><span class="dl">'</span><span class="k">sqlite3-ruby</span><span class="dl">'</span></span>, <span class="sy">:require</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">sqlite3</span><span class="dl">'</span></span>
</pre></div>
</div><br></p>
<p>And then use bundler to install it if necessary:
<div class="CodeRay">
  <div class="code"><pre>$ bundle install
Using rake (0.8.7) 
Using abstract (1.0.0) 
Using activesupport (3.0.0)
etc... </pre></div>
</div><br></p>
<p>And now finally I can start up my mongrel server
<div class="CodeRay">
  <div class="code"><pre>$ rails server mongrel
=> Booting Mongrel
=> Rails 3.0.0 application starting in development on http://0.0.0.0:3000
=> Call with -d to detach
=> Ctrl-C to shutdown server</pre></div>
</div><br></p>
<p>... and stream all 100,000 user records by opening http://localhost:3000/users.text. For this example, the users.txt file was about 5MB large... certainly there&rsquo;s no good reason to allocate over 200MB on the server to generate this!</p>
