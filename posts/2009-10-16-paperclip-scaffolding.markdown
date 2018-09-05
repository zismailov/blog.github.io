title: Paperclip scaffolding
date: 2009/10/16
tag: View Mapper

<p>I just updated the <a href="http://patshaughnessy.net/view_mapper">View Mapper gem</a> to support Paperclip. You can use it to generate scaffolding code that supports uploading and downloading Paperclip file attachments.</p>
<p><b>Creating a view for an existing model</b></p>
<p>If you have a model like this:</p>
<pre>class Song &lt; ActiveRecord::Base
  has_attached_file :mp3
end</pre>
<p>&hellip; you can generate a &ldquo;Paperclip view&rdquo; for this model like this:</p>
<pre>script/generate view_for song --view paperclip</pre>
<p>This will generate a controller, view and other code files that support uploading and downloading files. If you run your app you&rsquo;ll see the typical scaffolding user interface but with a file field for the &ldquo;mp3&rdquo; Paperclip attachment:</p>
<p><img src="http://patshaughnessy.net/assets/2009/10/16/new_song.png"/></p>
<p>View Mapper has:
<ul>
  <li>inspected your model (&ldquo;Song&rdquo; in this example) and found its Paperclip attachments and other standard ActiveRecord attributes</li>
  <li>called the Rails scaffold generator and passed in the ActiveRecord columns it found</li>
  <li>added additional view code to support Paperclip  (e.g. set the form to use multipart/form-data encoding)</li>
  <li>created a file field in the form for each Paperclip attachment (&ldquo;mp3&rdquo; in this example), as well as a link to each attachment in the show view code file.</li></ul>
<p>If you&rsquo;re not very familiar with Paperclip and how to use it or if you just want to get a Rails upload form working very quickly, then View Mapper can help you.</p>
<p><b>Creating an entirely new Paperclip model and view</b></p>
<p>View Mapper also provides a generator called &ldquo;scaffold_for_view&rdquo; that is identical to the standard Rails scaffold generator, except it will create the specified view. As an example, let&rsquo;s create a new Rails app from scratch that uses Paperclip; you should be able to type in these precise commands on your machine and get this example to work.</p>
<p>First, let&rsquo;s install View Mapper and create a new Rails app to display my MP3 library online (ignoring copyright issues for now):</p>
<pre>$ gem sources -a http://gemcutter.org
http://gemcutter.org added to sources
$ sudo gem install view_mapper
Successfully installed view_mapper-0.2.0
1 gem installed
Installing ri documentation for view_mapper-0.2.0...
Installing RDoc documentation for view_mapper-0.2.0...
$ rails music
      create  
      create  app/controllers
      create  app/helpers
      create  app/models
      create  app/views/layouts
etc&hellip;</pre>
<p>And now we can generate a new &ldquo;Song&rdquo; model that has a Paperclip attachment called &ldquo;MP3&rdquo; using View Mapper like this:</p>
<pre>$ cd music
$ ./script/generate scaffold_for_view song name:string artist:string
                    album:string play_count:integer --view paperclip:mp3
       error  The Paperclip plugin does not appear to be installed.</pre>
<p>Wait&hellip; I forgot to install Paperclip; let&rsquo;s do that and then try again:</p>
<pre>$ ./script/plugin install git://github.com/thoughtbot/paperclip.git
Initialized empty Git repository in /Users/pat/rails-apps/music/vendor/plugins/paperclip/.git/
remote: Counting objects: 71, done.
remote: Compressing objects: 100% (59/59), done.
remote: Total 71 (delta 7), reused 29 (delta 3)
Unpacking objects: 100% (71/71), done.
From git://github.com/thoughtbot/paperclip
 * branch            HEAD       -&gt; FETCH_HEAD
$ ./script/generate scaffold_for_view song name:string artist:string
                    album:string play_count:integer --view paperclip:mp3
      exists  app/models/
      exists  app/controllers/
      exists  app/helpers/
      create  app/views/songs
      exists  app/views/layouts/
      exists  test/functional/
etc&hellip;</pre>
<p>Finally, we just need to run db:migrate &ndash; one minor detail here is that the scaffold_for_view generator included the Paperclip columns (&ldquo;mp3_file_name,&rdquo; &ldquo;mp3_content_type,&rdquo; etc&hellip;) in the migration file to create the songs table:</p>
<pre>$ rake db:migrate
(in /Users/pat/rails-apps/music)
==  CreateSongs: migrating ====================================================
-- create_table(:songs)
   -&gt; 0.0022s
==  CreateSongs: migrated (0.0024s) ===========================================</pre>
<p>Now you can run your app and see the scaffolding UI I showed above, and will be able to upload and download MP3 files using Paperclip.</p>
<p>Let&rsquo;s take a quick look at exactly what is different about the scaffolding code View Mapper generated vs. the standard Rails scaffolding code:</p>
<pre>&lt;h1&gt;New song&lt;/h1&gt;

&lt;% form_for(@song<b>, :html =&gt; { :multipart =&gt; true }</b>) do |f| %&gt;
  &lt;%= f.error_messages %&gt;
  &lt;p&gt;
    &lt;%= f.label :name %&gt;&lt;br /&gt;
    &lt;%= f.text_field :name %&gt;
  &lt;/p&gt;

&hellip; etc &hellip;

<b>  &lt;p&gt;
    &lt;%= f.label :mp3 %&gt;&lt;br /&gt;
    &lt;%= f.file_field :mp3 %&gt;
  &lt;/p&gt;</b>
  &lt;p&gt;
    &lt;%= f.submit &#x27;Create&#x27; %&gt;
  &lt;/p&gt;
&lt;% end %&gt;

&lt;%= link_to &#x27;Back&#x27;, songs_path %&gt;</pre>
<p>The code in bold was generated by View Mapper specifically to support Paperclip since we used the &ldquo;--view paperclip&rdquo; command line option. You can see that &ldquo;:html =&gt; { :multipart =&gt; :true }&rdquo; was added to form_for to allow for file uploads, and also a file_field was added for the mp3 Paperclip attachment.</p>
<p>If you take a look at the show view, you&rsquo;ll see:</p>
<pre>&lt;p&gt;
  &lt;b&gt;Name:&lt;/b&gt;
  &lt;%=h @song.name %&gt;
&lt;/p&gt;

&hellip; etc &hellip;

<b>&lt;p&gt;
  &lt;b&gt;Mp3:&lt;/b&gt;
  &lt;%= link_to @song.mp3_file_name, @song.mp3.url %&gt;&lt;br&gt;
&lt;/p&gt;</b>

&lt;%= link_to &#x27;Edit&#x27;, edit_song_path(@song) %&gt; |
&lt;%= link_to &#x27;Back&#x27;, songs_path %&gt;</pre>
<p>Here a link to the file attachment was added, using Paperclip to provide the name and URL of the attachment.</p>
<p>Next I&rsquo;ll be adding support for nested attributes and complex forms to View Mapper.</p>
