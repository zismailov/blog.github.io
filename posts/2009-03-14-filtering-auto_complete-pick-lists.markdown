title: Filtering auto_complete pick lists
date: 2009/03/14
tag: the auto_complete plugin

<p>I&rsquo;ve written a lot here during the past few months about the auto_complete plugin and <a href="http://patshaughnessy.net/2009/1/30/repeated_auto_complete-changes-merged-into-auto_complete">how to get it to work with repeated text fields on a complex form</a>. Back in January <a href="http://patshaughnessy.net/2009/1/30/sample-app-for-auto-complete-on-a-complex-form">I modified Ryan Bates&rsquo;s complex forms sample app to illustrate how to use my version of the auto complete plugin to handle repeated text fields</a>. Here&rsquo;s what the form looks like in that sample app:</p>
<p><img src="http://patshaughnessy.net/assets/2009/3/14/unfiltered.png"></p>
<p>Here as the user types into a &ldquo;Task&rdquo; text field, a list of all of the existing task names in the system that match the characters typed in by the user are displayed in a drop down list. But what if I didn&rsquo;t want to display all of the matching task names? What if I wanted to display only the tasks for a given project? Or if I wanted to filter the task names in some other way based on other field values?</p>
<p>In this simple example, what if I only wanted to display Tasks 2a, 2b and 2c, since they belonged to Project Two?</p>
<p>Today I took a look at this problem and expected to see a number of simple solutions, but instead I was surprised to find that it is fairly difficult to do this. I got started by reading <a href="http://blog.andrewng.com/2008/09/08/autocomplete-with-multiple-related-form-fields-in-rails">this nice solution from Andrew Ng</a> (nice work Andrew!). Andrew explains how to get the Prototype javascript code to pass an additional HTTP parameter to the server when the user types into an autocomplete field. This additional parameter can then be used to filter the list of auto complete options differently. I&rsquo;ll let you read the details, but basically Andrew found that you can use a Javascript callback function like this to load a value from another field on your form, and pass it to the server in the Ajax request as an additional query string parameter:</p>
<pre>&lt;script type=&quot;text/javascript&quot;&gt;
  new Ajax.Autocompleter(
    &#x27;task_name&#x27;, 
    &#x27;task_name_auto_complete&#x27;, 
    &#x27;/projects/auto_complete_for_task_name&#x27;, 
    { callback: function(e, qs) {
        return qs + &#x27;&amp;project=&#x27; + $F(&#x27;project_name&#x27;);
      }
    }
  );
&lt;/script&gt;</pre>
<p>(I&rsquo;ve renamed the variables to use my project/tasks example.) What I didn&rsquo;t like about this was the need to manually code all of this javascript; there must be a way to get the auto_complete plugin to do this instead&hellip; and there is! If you look at the definition of text_field_with_auto_complete in <a href="http://github.com/patshaughnessy/auto_complete/blob/0814a25a754a235c5cf6f7a258fa405059a5ca6f/lib/auto_complete_macros_helper.rb">auto_complete_macros_helper.rb</a>, you&rsquo;ll see that it takes both tag_options and completion_options as parameters, and eventually calls auto_complete_field with the completion_options. Here&rsquo;s what auto_complete_field looks like in the auto_complete plugin:</p>
<pre>def auto_complete_field(field_id, options = {})
  function =  &quot;var #{field_id}_auto_completer = new Ajax.Autocompleter(&quot;
  function &lt;&lt; &quot;&#x27;#{field_id}&#x27;, &quot;

etc...

  js_options = {}
  js_options[:tokens] = etc...
  <b>js_options[:callback]   =
    &quot;function(element, value) { return #{options[:with]} }&quot; if options[:with]</b>
  js_options[:indicator]  = etc...
  js_options[:select]     = etc...
  js_options[:paramName]  = etc...

etc...

  function &lt;&lt; (&#x27;, &#x27; + options_for_javascript(js_options) + &#x27;)&#x27;)
  javascript_tag(function)
end</pre>
<p>If you look closely at the line I bolded above, you&rsquo;ll see that we can actually generate Andrew&rsquo;s Javascript callback function automatically by simply passing in a value for the &ldquo;with&rdquo; completion option when we call text_field_with_auto_complete in our view, like this:</p>
<pre>text_field_with_auto_complete :task, :name, {},
  {
    :method =&gt; :get,
    <b>:with =&gt;&quot;value + &#x27;&amp;project=&#x27; + $F(&#x27;project_name&#x27;)&quot;</b>
  }</pre>
<p>Again, this line of Javascript code is called when the user types into the task name field, and appends &ldquo;&amp;project=XYZ&rdquo; to the query string for the Ajax request. &ldquo;XYZ&rdquo; is the name of the project typed in by the user on the same form, loaded with prototype&rsquo;s &ldquo;$F&rdquo; (Form element get value) function. The &ldquo;:method =&gt; :get&rdquo; completion option is used to avoid problems with CSRF protection; see <a href="http://www.ruby-forum.com/topic/128970">http://www.ruby-forum.com/topic/128970</a>. If you look at your server&rsquo;s log file, you&rsquo;ll see HTTP requests that look something like this now:</p>
<pre>127.0.0.1 - - [13/Mar/2009:16:17:03 EDT] &quot;
GET /projects/auto_complete_for_task_name?task%5Bname%5D=T&amp;project=Project%20Two
HTTP/1.1&quot; 200 57</pre>
<p>Here we can see the &ldquo;auto_complete_for_task_name&rdquo; route is called and given two request parameters: &ldquo;task[name]&rdquo; and &ldquo;project&rdquo;. The task name is the standard parameter generated by the autocompleter javascript, and &ldquo;project&rdquo; is the additional parameter created by the callback function generated by the :with option.</p>
<p>Now&hellip; how do we handle the &ldquo;project&rdquo; parameter in our controller code? Without modifying the auto_complete plugin itself, you would have to write your own controller method and not use the &ldquo;auto_complete_for&rdquo; macro at all. <a href="http://blog.andrewng.com/2008/09/08/autocomplete-with-multiple-related-form-fields-in-rails">Andrew shows how to do this on his blog.</a> What I want to explore here now is whether there&rsquo;s a way to change the auto_complete_for method to allow for customizations of the query used to load the auto complete options.</p>
<p>To understand the problem a bit better, let&rsquo;s take a look at how &ldquo;auto_complete_for&rdquo; is implemented in the auto_complete plugin:</p>
<pre>def auto_complete_for(object, method, options = {})
  define_method(&quot;auto_complete_for_#{object}_#{method}&quot;) do
    find_options = { 
      :conditions =&gt; [ &quot;LOWER(#{method}) LIKE ?&quot;, &#x27;%&#x27; +
        params[object][method].downcase + &#x27;%&#x27; ], 
      :order =&gt; &quot;#{method} ASC&quot;,
      :limit =&gt; 10 }.merge!(options)
    @items = object.to_s.camelize.constantize.find(:all, find_options)
    render :inline =&gt; &quot;&lt;%= auto_complete_result @items, &#x27;#{method}&#x27; %&gt;&quot;
  end	
end</pre>
<p>When this is called as your Rails application initializes, it adds a new method to your controller called something like &ldquo;auto_complete_for_task_name&rdquo; with your model and column names instead. What we want to do is filter the query results differently, by using a new HTTP parameter &ndash; so we need to modify the &ldquo;conditions&rdquo; hash passed into find :all. At first I tried to do this by passing in different values for the &ldquo;options&rdquo; parameter, since that&rsquo;s merged with the default options and then passed into find :all. However, the problem with this approach is that whatever you pass in using &ldquo;options&rdquo; will not have access to the request parameters, since it&rsquo;s passed in when the controller is initialized, and not when the HTTP request is received.</p>
<p>So the solution is to pass in a block that is evaluated when the request is received, and when the generated method is actually called. I wrote a variation on auto_complete_for called &ldquo;filtered_auto_complete_for:&rdquo;</p>
<pre>def filtered_auto_complete_for(object, method)
  define_method(&quot;auto_complete_for_#{object}_#{method}&quot;) do
    find_options = { 
      :conditions =&gt; [ &quot;LOWER(#{method}) LIKE ?&quot;, &#x27;%&#x27; +
        params[object][method].downcase + &#x27;%&#x27; ], 
      :order =&gt; &quot;#{method} ASC&quot;,
      :limit =&gt; 10 }
    <b>yield find_options, params</b>
    @items = object.to_s.camelize.constantize.find(:all, find_options)
    render :inline =&gt; &quot;&lt;%= auto_complete_result @items, &#x27;#{method}&#x27; %&gt;&quot;
  end
end</pre>
<p>Filtered_auto_complete_for takes a block and evaluates it when the actual HTTP Ajax request is received from the auto complete Javascript. The block is provided with the find options hash and also the request parameters. This enables the controller&rsquo;s block to modify the find options in any way it would like, possibly using the HTTP request parameters provided. I&rsquo;ve also removed the options parameter since that&rsquo;s not necessary any more.</p>
<p>As an example, here&rsquo;s my sample app&rsquo;s controller code:</p>
<pre>class ProjectsController &lt; ApplicationController

  # Handle auto complete for project names as usual:
  auto_complete_for :project, :name

  # For task name auto complete, only display tasks
  # that belong to the given project: 
  filtered_auto_complete_for :task, :name do | find_options, params|
    find_options.merge!(
      {
        :include =&gt; :project,
        :conditions =&gt; [ &quot;LOWER(tasks.name) LIKE ? AND projects.name = ?&quot;,
                         &#x27;%&#x27; + params[&#x27;task&#x27;][&#x27;name&#x27;].downcase + &#x27;%&#x27;,
                         params[&#x27;project&#x27;] ],
        :order => "tasks.name ASC"
      }
    )
  end

  def index
    @projects = Project.find(:all)
  end

  etc...</pre>
<p>The code in this sample block modifies the find_options by adding &ldquo;:include =&gt; :project&rdquo;. This causes ActiveRecord to use a JOIN to include columns from the project table in the query (in this sample app Project has_many Tasks, and each Task belongs_to a Project). Then it matches on the project name, in addition to the portion of the task name typed in by the user so far. This limits the auto complete values to just the tasks that belong to the given project:</p>
<p><img src="http://patshaughnessy.net/assets/2009/3/14/filtered.png"></p>
<p>When I have time during the next few days I&rsquo;ll add &ldquo;filtered_auto_complete_for&rdquo; to my forked version of the auto_complete plugin&hellip; first I need to write some unit tests for it, and be sure it works as intended.  After that, I&rsquo;ll post this sample app back on github and you can try it yourself.</p>
