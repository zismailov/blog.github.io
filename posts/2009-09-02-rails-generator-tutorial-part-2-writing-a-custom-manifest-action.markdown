title: "Rails generator tutorial part 2: writing a custom manifest action"
date: 2009/09/02
tag: Ruby

<p>In <a href="http://patshaughnessy.net/2009/8/23/tutorial-how-to-write-a-rails-generator">part one of this tutorial</a> I wrote a simple rails generator that creates a controller code file in app/controllers. This time I want to finish up my simple &ldquo;VersionGenerator&rdquo; example by enabling it to add a line to the routes.rb file. In other words, I want to be able to use a manifest action called &ldquo;route&rdquo; like this:</p>
<pre>def manifest
    record do |m|
      m.template(&#x27;controller.rb&#x27;, &#x27;app/controllers/version_controller.rb&#x27;)
      <b>m.route :name =&gt; &#x27;version&#x27;,
              :controller =&gt; &#x27;version&#x27;,
              :action =&gt; &#x27;display_version&#x27;</b>
    end
  end</pre>
<p>The problem for today is how to implement the &ldquo;route&rdquo; action, which is not supported by Rails by default. The Rails generator base class code does provide a few other actions you can use in your manifest, like &ldquo;file&rdquo; or &ldquo;directory,&rdquo; which create a file or directory. Rails even provides a method called &ldquo;route_resources&rdquo; to add a &ldquo;map.resources&rdquo; line to routes.rb, but not a simple named route line like what we need for this example.</p>
<p>This specific action might be useful for you, but my real goal is to explore how Rails generators work in general and discuss some of the issues you might run into if you need a custom manifest action like this for any purpose. Let&rsquo;s get started by just adding the new action method we need right inside our generator class&hellip;</p>
<pre>def route(route_options)
  puts &quot;This will create a new route!&quot;
end</pre>
<p>&hellip;and see what happens when we run script/generate with manifest method above:</p>
<pre>$ ./script/generate version ../build-info/version.txt 
   identical  app/controllers/version_controller.rb
This will create a new route!</pre>
<p>Very promising! All we need to do is add the necessary code here and we&rsquo;ll be all set. But first let&rsquo;s take a closer look at what&rsquo;s happening by adding a call to &ldquo;caller&rdquo; inside this new method and see how it&rsquo;s being called by the Rails generator system:</p>
<pre>def route(route_options)
  puts &quot;This will create a new route!&quot;
  puts caller
end</pre>
<p>Here&rsquo;s what is displayed when we run this (paths shortened for readability):</p>
<pre>$ ./script/generate version ../build-info/version.txt
   identical  app/controllers/version_controller.rb
This will create a new route!
/usr/local/lib/ruby/1.8/delegate.rb:270:in `__send__&#x27;
/usr/local/lib/ruby/1.8/delegate.rb:270:in `method_missing&#x27;
/path/to/rails-2.3.2/lib/rails_generator/manifest.rb:47:in `send&#x27;
/path/to/rails-2.3.2/lib/rails_generator/manifest.rb:47:in `send_actions&#x27;
/path/to/rails-2.3.2/lib/rails_generator/manifest.rb:46:in `each&#x27;
/path/to/rails-2.3.2/lib/rails_generator/manifest.rb:46:in `send_actions&#x27;
/path/to/rails-2.3.2/lib/rails_generator/manifest.rb:31:in `replay&#x27;
/path/to/rails-2.3.2/lib/rails_generator/commands.rb:42:in `invoke!&#x27;
<b>/path/to/rails-2.3.2/lib/rails_generator/scripts/../scripts.rb:31:in `run&#x27;</b>
/path/to/rails-2.3.2/lib/commands/generate.rb:6
/usr/local/lib/ruby/site_ruby/1.8/rubygems/custom_require.rb:31:in `gem_original_require&#x27;
/usr/local/lib/ruby/site_ruby/1.8/rubygems/custom_require.rb:31:in `require&#x27;
./script/generate:3</pre>
<p>There&rsquo;s a lot going on here; in fact, when I started to learn more about Rails generators and how they work I was shocked at how complicated their object model and design are. I won&rsquo;t try to explain all of it here, but let&rsquo;s take a closer look at the one line I bolded above: line 31 of <a href="http://github.com/rails/rails/blob/a147becfb86b689ab25e92edcfbb4bcc04108099/railties/lib/rails_generator/scripts.rb#L31">lib/rails_generator/scripts.rb</a></p>
<pre>Rails::Generator::Base.instance(options[:generator], args, options)
                      .command(options[:command]).invoke!</pre>
<p>This is called when you run script/generate, by the &ldquo;Rails::Generator::Scripts::Generate&rdquo; class. Translating this line of Ruby into English, we get: &ldquo;Create an instance of the specified generator, passing in the arguments and options provided on the command line; then create an instance of the specified command that uses that generator; then invoke the command.&rdquo;</p>
<p>What this means is that actually Rails generator classes do not do the real work of generating code &ndash; Rails generator &ldquo;commands&rdquo; do! The <a href="http://github.com/rails/rails/blob/a147becfb86b689ab25e92edcfbb4bcc04108099/railties/lib/rails_generator/commands.rb">lib/rails_generator/commands.rb</a> file defines a series of command objects inside the Rails::Generator::Commands module. The two most important command classes are Create and Destroy.  The &ldquo;Create&rdquo; command is called when you execute script/generate, like what happened in the stack trace above. This class contains the implementation of the &ldquo;template&rdquo; method we used before, plus various other methods like &ldquo;file&rdquo; and &ldquo;directory&rdquo; that create files and directories, etc.</p>
<p>The &ldquo;Destroy&rdquo; command is called when you execute script/destroy from the command line (I removed &ldquo;puts caller&rdquo; first before running this):</p>
<pre>$ ./script/destroy version ../build-info/version.txt 
This will create a new route!
          rm  app/controllers/version_controller.rb</pre>
<p>This class has all of the same methods that the Create class does, except that they perform the opposite action: instead of creating a file or directory, they delete them. The Destroy class also executes the actions in the opposite order, by looping backwards through the manifest that we recorded in our generator. Here the destroy command has deleted our version_controller.rb file we generated earlier. Note that it still displays &ldquo;This will create a new route!&rdquo;</p>
<p>If you&rsquo;re interesting in diving into the real details about how Rails generators work, then read the code in <a href="http://github.com/rails/rails/blob/a147becfb86b689ab25e92edcfbb4bcc04108099/railties/lib/rails_generator/commands.rb">commands.rb</a> very carefully. In particular, look at how the &ldquo;self.included&rdquo; and &ldquo;self.instance&rdquo; methods at the top work; these methods are used in the line I showed above to create the specified command, supplying the specified generator as an argument to the command constructor. The &ldquo;invoke!&rdquo; method on commands actually plays back the recorded manifest. Also all of the actions that are available to your generator's manifest method are defined in this file. One other interesting detail I don&rsquo;t have space to explain carefully here is that the command objects contain the corresponding generator class as a delegate object; in other words they contain an instance of the generator as a member variable:</p>
<pre># module Rails::Generator::Commands...
class Base &lt; DelegateClass(Rails::Generator::Base)</pre>
<p>This explains the top two lines in the stack trace above:</p>
<pre>/usr/local/lib/ruby/1.8/delegate.rb:270:in `__send__&#x27;
/usr/local/lib/ruby/1.8/delegate.rb:270:in `method_missing&#x27;</pre>
<p>&ldquo;DelegateClass&rdquo; is defined by delegate.rb, which is a <a href="http://www.ruby-doc.org/stdlib/libdoc/delegate/rdoc/index.html">Ruby library that implements the delegate pattern</a>. This is why we were able to add the new &ldquo;route&rdquo; method right in our generator; when Ruby found the &ldquo;route&rdquo; method missing in the Create command, it delegated the call to our generator object.</p>
<p>Obviously we have a problem here: our new &ldquo;route&rdquo; method needs to be able to remove a route from routes.rb if the Destroy command is called, as well as create a new one for the Create command. One way to implement our new &ldquo;route&rdquo; method would be check the &ldquo;options[:command]&rdquo; value that we saw above on <a href="http://github.com/rails/rails/blob/a147becfb86b689ab25e92edcfbb4bcc04108099/railties/lib/rails_generator/scripts.rb#L31">line 31 of scripts.rb</a>, like this:</p>
<pre>def route(route_options)
  if options[:command] == :create
    puts &quot;This will add a new route to routes.rb.&quot;
  elsif options[:command] == :destroy
    puts &quot;This will remove the new route from routes.rb.&quot;
  end
end</pre>
<p>Now if we run script/generate again, our new method will create a route:</p>
<pre>$ ./script/generate version ../build-info/version.txt
      create  app/controllers/version_controller.rb
This will add a new route to routes.rb.</pre>
<p>And if we run the destroy command, we&rsquo;ll remove the route:</p>
<pre>$ ./script/destroy version ../build-info/version.txt 
This will remove the new route from routes.rb.
          rm  app/controllers/version_controller.rb</pre>
<p>It turns out a cleaner way to implement the &ldquo;route&rdquo; method is to directly add the method to both the Create and Destroy command classes. This allows me to call a utility method called &ldquo;gsub_file&rdquo; in the Rails::Generator::Commands::Base class which I wouldn&rsquo;t have direct access to from my generator class. It also avoids the somewhat ugly if statement on the options[:command] value, and finally it might make it easier for me someday to refactor the new route methods into a separate module that I could use with various different generators that might need to add and remove routes.</p>
<p>Anyway, here&rsquo;s the finished code for the entire generator:</p>
<pre>class VersionGenerator &lt; Rails::Generator::NamedBase
  attr_reader :version_path
  def initialize(runtime_args, runtime_options = {})
    super
    @version_path = File.join(RAILS_ROOT, name)
  end
  def manifest
    record do |m|
      m.template(&#x27;controller.rb&#x27;, &#x27;app/controllers/version_controller.rb&#x27;)
      m.route :name =&gt; &#x27;version&#x27;,
              :controller =&gt; &#x27;version&#x27;,
              :action =&gt; &#x27;display_version&#x27;
    end
  end
end

module Rails
  module Generator
    module Commands

      class Base
        def route_code(route_options)
          &quot;map.#{route_options[:name]} &#x27;#{route_options[:name]}&#x27;, :controller =&gt; &#x27;#{route_options[:controller]}&#x27;, :action =&gt; &#x27;#{route_options[:action]}&#x27;&quot;
        end
      end

# Here's a readable version of the long string used above in route_code;
# but it should be kept on one line to avoid inserting extra whitespace
# into routes.rb when the generator is run:
# &quot;map.#{route_options[:name]} &#x27;#{route_options[:name]}&#x27;,
#     :controller =&gt; &#x27;#{route_options[:controller]}&#x27;,
#     :action =&gt; &#x27;#{route_options[:action]}&#x27;&quot;

      class Create
        def route(route_options)
          sentinel = &#x27;ActionController::Routing::Routes.draw do |map|&#x27;
          logger.route route_code(route_options)
          gsub_file &#x27;config/routes.rb&#x27;, /(#{Regexp.escape(sentinel)})/mi do |m|
              &quot;#{m}\n  #{route_code(route_options)}\n&quot;
          end
        end
      end

      class Destroy
        def route(route_options)
          logger.remove_route route_code(route_options)
          to_remove = &quot;\n  #{route_code(route_options)}&quot;
          gsub_file &#x27;config/routes.rb&#x27;, /(#{to_remove})/mi, &#x27;&#x27;
        end
      end

    end
  end
end</pre>
<p>More or less copied from the existing route_resources action found in <a href="http://github.com/rails/rails/blob/a147becfb86b689ab25e92edcfbb4bcc04108099/railties/lib/rails_generator/commands.rb">commands.rb</a>, this works as follows: If we call script/generate then the route action method in the Create command class is called. This uses gsub_file to look for the &ldquo;sentinel&rdquo; or target area of the file and replaces it with the sentinel + the new route code. I also use the &ldquo;logger&rdquo; method to display log messages using the Rails::Generator::SimpleLogger class. Here&rsquo;s what it looks like on the command line:</p>
<pre>$ ./script/generate version ../build-info/version.txt
      create  app/controllers/version_controller.rb
       route  map.version &#x27;version&#x27;,
                 :controller =&gt; &#x27;version&#x27;,
                 :action =&gt; &#x27;display_version&#x27;</pre>
<p>If script/destroy is called, then the second route implementation in the Destroy class is called. This uses gsub_file to remove the route code. Finally, the &ldquo;route_code&rdquo; method returns the route code that we want to generate or remove from routes.rb. This time on the command line we get:</p>
<pre>$ ./script/destroy version ../build-info/version.txt 
remove_route  map.version 'version',
                 :controller => 'version',
                 :action => 'display_version'
          rm  app/controllers/version_controller.rb</pre>
