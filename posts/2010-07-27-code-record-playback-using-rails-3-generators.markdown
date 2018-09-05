title: Code record/playback using Rails 3 generators
date: 2010/07/27
url: /2010/7/27/code-record-playback-using-rails-3-generators
tag: Ruby

<p>For a while I&rsquo;ve been thinking that writing a Rails generator is a fairly difficult thing to do. First you need to learn about Thor and the Rails generator system: what sort of Ruby class you need to write, how to handle arguments, how to run commands like &ldquo;copy_file&rdquo;, etc. Then you need to write ERB files to produce the code that you&rsquo;d like to generate, which is always a chore.</p>
<p>So last night I wrote a gem called <a href="http://github.com/patshaughnessy/generate_from_diff">generate_from_diff</a> that let&rsquo;s you create Rails 3 generators automatically using a code record/playback model. Here&rsquo;s how it works:
  <ul>
    <li>You make a code change in some Rails project, and commit it to your Git repo.</li>
    <li>You run a command from my gem to extract this code change from the Git repo into a new Rails 3 generator.</li>
    <li>You later run this new generator in any other Rails 3 app, and your recorded code changes are &ldquo;played back,&rdquo; or applied to your new project, using the Unix patch utility written by Larry Wall.</li>
  </ul>
</p>
<p>You&rsquo;ve created a Rails 3 generator without ever writing a single line of generator code!</p>
<p>Disclaimer: I just got this working last night, so it&rsquo;s still very rough; but if the idea seems worthwhile I&rsquo;ll clean it up and try to make it more robust and useable. Another disclaimer: none of this will work on Windows since it relies on the Unix patch utility.</p>
<p><b>Example: recording code into a new generator</b></p>
<p>Let&rsquo;s look at an example to try to make this a bit clearer. Support I create a new Rails 3 application:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails new first_app
      <span class="s">create</span>  
      <span class="s">create</span>  README
      <span class="s">create</span>  Rakefile
      <span class="s">create</span>  config.ru
      <span class="s">create</span>  .gitignore
      <span class="s">create</span>  Gemfile
etc...</pre></div>
</div><br>
<p>And let&rsquo;s run bundle install to be sure I have all the required gems. This is usually not necessary for a new, empty Rails app, but I want to have my Gemfile.lock file created... more on that in a moment.</p>
<div class="CodeRay">
  <div class="code"><pre>$ cd first_app
$ bundle install
Fetching source index for http://rubygems.org/
Using rake (0.8.7) 
Using abstract (1.0.0) 
Using activesupport (3.0.0.beta4) 
Using builder (2.1.2) 
etc...</pre></div>
</div><br>
<p>And let&rsquo;s create a new Git repo here and check the empty application into it:</p>
<div class="CodeRay">
  <div class="code"><pre>$ git init
Initialized empty Git repository in /Users/pat/.../first_app/.git/
$ git add .
$ git commit -m&quot;New sample app&quot;</pre></div>
</div><br>
<p>This first Git revision will serve as the baseline for recording my new generator, which I&rsquo;ll do in a minute. The reason I ran bundle install was to insure that the Gemfile.lock file would be included in the baseline... and so not included in the recorded code change.</p>
<p>Now let&rsquo;s write some code that I can record into a new generator. Suppose at my company I want to create a controller that returns the build number, diagnostics and some other information about each of my Rails apps. I might do this by creating a new controller as follows:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails generate controller build_info
      <span class="s">create</span>  app/controllers/build_info_controller.rb
      invoke  erb
      <span class="s">create</span>    app/views/build_info
      invoke  test_unit
      <span class="s">create</span>    test/functional/build_info_controller_test.rb
      invoke  helper
etc...</pre></div>
</div><br>
<p>And in this new controller I&rsquo;ll add a single index action:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">BuildInfoController</span> &lt; <span class="co">ApplicationController</span>
  <span class="r">def</span> <span class="fu">index</span>
    render <span class="sy">:text</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">Some interesting build info about this app...</span><span class="dl">'</span></span>
  <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br>
<p>Finally, I&rsquo;ll add a route to send &ldquo;build_info&rdquo; requests to this action:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="co">FirstApp</span>::<span class="co">Application</span>.routes.draw <span class="r">do</span> |map|

  match <span class="s"><span class="dl">'</span><span class="k">build_info</span><span class="dl">'</span></span> =&gt; <span class="s"><span class="dl">'</span><span class="k">build_info#index</span><span class="dl">'</span></span>

...etc...
</pre></div>
</div><br>
<p>This is somewhat silly, but it&rsquo;s simple enough to use as an example here. Now if I run my app I&rsquo;ll get this fascinating page:</p>
<p><img src="http://patshaughnessy.net/assets/2010/7/27/Picture_1.png"/></p>
<p>Next let&rsquo;s &ldquo;record&rdquo; this sample code by using generate_from_diff to create a new Rails generator for it. First, we need to install generate_from_diff:</p>
<div class="CodeRay">
  <div class="code"><pre>$ gem install generate_from_diff
Successfully installed generate_from_diff-0.0.1
1 gem installed
Installing ri documentation for generate_from_diff-0.0.1...
Installing RDoc documentation for generate_from_diff-0.0.1...</pre></div>
</div><br>
<p>Next, let&rsquo;s commit my new controller and routes.rb code changes:</p>
<div class="CodeRay">
  <div class="code"><pre>$ git add .

$ git status
# On branch master
# Changes to be committed:
#   (use &quot;git reset HEAD &lt;file&gt;...&quot; to unstage)
#
#        <span class="s">new file:   app/controllers/build_info_controller.rb</span>
#        <span class="s">new file:   app/helpers/build_info_helper.rb</span>
#        <span class="s">modified:   config/routes.rb</span>
#        <span class="s">new file:   test/functional/build_info_controller_test.rb</span>
#        <span class="s">new file:   test/unit/helpers/build_info_helper_test.rb</span>
#

$ git commit -m&quot;Build info&quot;
Created commit 037ca3b: Build info
 5 files changed, 22 insertions(+), 0 deletions(-)
 create mode 100644 app/controllers/build_info_controller.rb
 create mode 100644 app/helpers/build_info_helper.rb
 create mode 100644 test/functional/build_info_controller_test.rb
 create mode 100644 test/unit/helpers/build_info_helper_test.rb</pre></div>
</div><br>
<p>One last detail: we need to edit the Gemfile to load generate_from_diff into this application:</p>
<div class="CodeRay">
  <div class="code"><pre>source <span class="s"><span class="dl">'</span><span class="k">http://rubygems.org</span><span class="dl">'</span></span>

<div class='container'>gem <span class="s"><span class="dl">'</span><span class="k">generate_from_diff</span><span class="dl">'</span></span><span class='overlay'></span></div>
gem <span class="s"><span class="dl">'</span><span class="k">rails</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">3.0.0.beta4</span><span class="dl">'</span></span>
etc...</pre></div>
</div>
<br>
<p>Finally we create our new generator by just running this command:</p>
<p>
  <div class="CodeRay">
    <div class="code"><pre>$ rails generate generator_from_diff build_info HEAD~1 HEAD
    <span class="s">create</span>  lib/generators/build_info
    <span class="s">create</span>  lib/generators/build_info/build_info_generator.rb
    <span class="s">create</span>  lib/generators/build_info/USAGE
       <span class="s">run</span>  git diff --no-prefix HEAD~1 HEAD from &quot;.&quot;</pre></div>
  </div><br>
<p>Ok - what happened here was that I ran a generator called &ldquo;generator_from_diff&rdquo; that is located inside the generate_from_diff gem. I provided it with the name of the new generator I want to create: &ldquo;build_info&rdquo; in this example. This is similar to how the Rails 3 &ldquo;generator generator&rdquo; works: it generates a generator. But next I provide two Git revisions, in this example &ldquo;HEAD~1&rdquo; and &ldquo;HEAD,&rdquo; the first and second revisions in my Git repo. The first value is the baseline revision: what to compare to. In this example, this is my new, empty Rails application. The second revision is what code to record and save into the new generator, in this example this revision contains all of my controller and routes.rb changes.</p>
<p><b>Example: playing back code using a generator</b></p>
<p>Now let&rsquo;s see if we can use this new Rails generator to copy the build_info controller and route into a different Rails app. First, let&rsquo;s create a second, new Rails application:</p>
<div class="CodeRay">
  <div class="code"><pre>$ cd ..
$ rails new second_app
    <span class="s">create</span>  
    <span class="s">create</span>  README
    <span class="s">create</span>  Rakefile
    <span class="s">create</span>  config.ru
...etc...
$ cd second_app</pre></div>
</div><br>
<p>And next, let&rsquo;s copy the new generator we just created in the first app, over to this new app:</p>
<div class="CodeRay">
  <div class="code"><pre>$ mkdir lib/generators
$ cp -r ../first_app/lib/generators/build_info lib/generators</pre></div>
</div><br>
<p>And now we can just run our new generator to playback the code changes that I recorded above:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails generate build_info
      <span class="s">gsub</span>  lib/generators/build_info/build_info.patch
       <span class="s">run</span>  patch -p0 &lt; /Users/pat/.../second_app/lib/generators/build_info/build_info.patch from &quot;.&quot;
  patching file app/controllers/build_info_controller.rb
  patching file app/helpers/build_info_helper.rb
  patching file config/routes.rb
  patching file test/functional/build_info_controller_test.rb
  patching file test/unit/helpers/build_info_helper_test.rb</pre></div>
</div><br>
<p>That&rsquo;s it! Now I can run the second app and see the same build status page that we had before:</p>
<p><img src="http://patshaughnessy.net/assets/2010/7/27/Picture_1.png"/></p>
<p><b>How does this actually work?</b></p>
<p>Here&rsquo;s what is going on under the hood. First, when you record your code changes into the new generator like this:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails generate generator_from_diff build_info HEAD~1 HEAD</pre></div>
</div><br>
<p>... the &ldquo;generator_from_diff&rdquo; code actually runs the &ldquo;git diff&rdquo; command like this:</p>
<div class="CodeRay">
  <div class="code"><pre>$ git diff HEAD~1 HEAD
diff --git a/app/controllers/build_info_controller.rb
           b/app/controllers/build_info_controller.rb
new file mode 100644
index 0000000..c44d83e
--- /dev/null
+++ b/app/controllers/build_info_controller.rb
@@ -0,0 +1,5 @@
+class BuildInfoController &lt; ApplicationController
+  def index

...etc...
</pre></div>
</div><br>
<p>This produces a list of all the text changes that were made from one revision (HEAD~1) to another (HEAD). These are then saved into a file called &ldquo;build_info.patch,&rdquo; saved inside the new generator.</p>
<p>Later, the text differences, the &ldquo;patch,&rdquo; are applied to whatever new or existing files are found relative to the current directory when you run the generator. This copies the new controller file as well as the new route inside of routes.rb into the other application. The patch file is applied using this command:</p>
<div class="CodeRay">
  <div class="code"><pre>patch -p0 &lt; lib/generators/build_info/build_info.patch</pre></div>
</div><br>
<p>I use patch instead of git apply to avoid the need to match revision id&rsquo;s; these will be different from one repo to another.</p>
<p><b>Ok sounds interesting - so where are you going with this next?</b></p>
<p>I think it&rsquo;s cool to be able to &ldquo;record&rdquo; Rails generators without writing any code. If this seems like a useful idea, then I&rsquo;ll spend some more time to clean it up and make it more robust. For example, I&rsquo;m thinking of adding some code to warn you before the patch is run if there are unexpected files present, or if some expected files are missing.</p>
<p>Next, I&rsquo;m considering enhancing the gem to perform search/replace using arguments or options that you specify when recording the generator. For example, suppose you recorded a series of code changes that had to do with a model called &ldquo;Person.&rdquo; But imagine that you want to be able to playback those code changes in a target application that might have a different model name, &ldquo;User&rdquo; instead of &ldquo;Person&rdquo; for example. Then the gem could search/replace on the patch file, both when it&rsquo;s recorded and again when it&rsquo;s played back, to cause the generated code to use User instead of Person.</p>
