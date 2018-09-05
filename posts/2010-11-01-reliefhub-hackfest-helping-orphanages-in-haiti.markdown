title: "ReliefHub hackfest: helping orphanages in Haiti"
date: 2010/11/1

<p>Last month I had a very unique experience: I took a week off from my day job to work with a team of volunteers to build &ldquo;ReliefHub,&rdquo; an online charity site designed to help orphanages in Haiti. It&rsquo;s still unfinished, but here&rsquo;s what the site looks like right now:</p>
<p><img src="/assets/images/reliefhub1.png"></p>
<p>After the terrible earthquake in January 2010, Rousseau Aurelien and James McElhiney started ReliefHub with this mission:</p>
<blockquote>ReliefHub is dedicated to working with orphanages around the world to help perpetuate their effectiveness and provide a foundation for gathering the resources they require to continue to care for millions of orphans and abandoned children.</blockquote>
<p>Initially the site will help orphanages in Haiti, but may expand it’s scope in the future. We’ve also put the <a href="http://github.com/ReliefHub/reliefhub">source code on github</a> and hope that someday it can become a general open source platform for online charities.</p>
<p>The ReliefHub business model was designed to be similar to <a href="http://www.donorschoose.org">Donor&rsquo;s Choose</a>, a successful online charity supporting school children and teachers across the country. The idea is that on the site you&rsquo;ll be able to see a list of orphanages:</p>
<p><img src="http://patshaughnessy.net/assets/2010/10/31/reliefhub2.png"></p>
<p>... and donate any amount of money towards one or more of their very specific needs. Once donations reach the target amount, ReliefHub&rsquo;s operations in Haiti will purchase the items and deliver them to the orphanage.</p>
<p>As you can see, in a week we were able to build the foundations of the web site, but the user interface still needs some polish and there are a variety of functional user stories that are still not complete. A team of developers from <a href="http://www.thoughtworks.com/">ThoughtWorks</a> is now continuing to work on the site. I&rsquo;ll post an update here with a link once the site is finished and live online.</p>
<p><b>If you&rsquo;d like to help out finishing the site</b>, just contact me and I&rsquo;ll put you in touch with ReliefHub. As I mentioned above, all of the code is open source: <a href="http://github.com/ReliefHub/reliefhub">http://github.com/ReliefHub/reliefhub</a>. ReliefHub needs both volunteer Rails developers and web designers to finish the site up.</p>
<h2>Volunteering: A way to work with great people and new ideas</h2>
<p>Aside from actually being able to do something to help the worsening situation in Haiti, the best part of the ReliefHub hackfest was having the chance to with a great group of people, using ideas and technologies that I had never used before.</p>
<p><table cellpadding="0" cellspacing="0" border="0">
  <tr><td><img src="http://patshaughnessy.net/assets/2010/10/31/reliefhub-team.jpg"></td></tr>
  <tr><td align="center"><small><i>Part of the team, left to right: Yan, Alex, James M., Dan and Rousseau</i></small></td></tr>
</table></p><br>
<p><a href="http://www.futurefridays.com/">Rousseau Aurelien</a> and <a href="http://www.futurefridays.com/">James McElhiney</a>: After the earthquake in January, James and Rousseau conceived of the ReliefHub idea. Rousseau has been working to find orphanages in Haiti for the site, while James did a lot of the technical planning and development work himself.</p>
<p><a href="http://www.futurefridays.com/">Tiffany Karl</a> and <a href="http://technologyforsocialinnovation.blogspot.com/">Amy Grandov</a> were our product owners - they collaborated ahead of time to come up with the user stories on our backlog.</p>
<p><a href="http://melissayasko.com/">Melissa Yasko</a> designed the site and <a href="http://www.mckinsey.com/">James Torio</a> converted Melissa&rsquo;s design into HTML/CSS.</p>
<p><a href="http://blog.newtonlabs.org/">Thomas Newton</a> and <a href="http://www.alexrothenberg.com/">Alex Rothenberg</a> worked together together to develop the rotating slide show of orphanage images for the home page using the <a href="http://ekallevig.com/jshowoff">Showoff JQuery plugin</a>. Alex also implemented the code for showing the orphanage photos across the site using <a href="http://github.com/thoughtbot/paperclip">Paperclip</a> and <a href="http://aws.amazon.com/s3/">S3</a>.
<p><a href="http://www.mckinsey.com/">Yanwing Wong</a> worked with <a href="http://www.futurefridays.com/">James McElhiney</a> to implement a login form integrated with Twitter and Facebook, using Janrain Engage with the Devise gem. See <a href="http://railscasts.com/episodes/233-engage-with-devise">Ryan Bates&rsquo; screen cast</a> for more info on this.</p>
<p><img src="http://patshaughnessy.net/assets/2010/10/31/reliefhub3.png"></p>
<p><a href="http://dancroak.com">Dan Croak</a> jumpstarted the project on Rails 3 using <a href="http://github.com/thoughtbot/suspenders">ThoughtBot&rsquo;s suspenders</a> and also setup a continuous integration server for us in a matter of just minutes using <a href="http://github.com/defunkt/cijoe">CI Joe</a> on an Ubuntu server graciously donated to us for the week by <a href="http://railsmachine.com/">Rails Machine</a>.</p>
<p>Dan also helped the team get up to speed on the latest testing tools and ideas from ThoughtBot, like <a href="http://github.com/thoughtbot/bourne">Bourne</a>, <a href="http://robots.thoughtbot.com/post/159805987/speculating-with-shoulda">Shoulda RSpec matchers:</a></p>
<div class="CodeRay">
  <div class="code"><pre>describe <span class="co">Project</span> <span class="r">do</span>
  let(<span class="sy">:project</span>) { Factory(<span class="sy">:project</span>) }

  it { should belong_to(<span class="sy">:organization</span>) }
  it { should have_many(<span class="sy">:donations</span>) }
<span class="r">end</span>
</pre></div>
</div><br/>
<p>... and <a href="http://robots.thoughtbot.com/post/284805810/gimme-three-steps">Factory Girl cucumber steps</a>:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">Scenario:</span> Visitor reviews project
  <span class="r">Given</span> the following organization exists:
    | name                          |
    | Mission des Eglises Baptistes |
  <span class="r">And</span> the following project exists:
    | name                | funds purpose           | organization              
    | Christmas Toy Drive | New or gently used toys | name: Mission des Eglis...
etc...
</pre></div>
</div><br/>
<p>Dan taught us a lot more than this, and also singlehandedly implemented a number of the user stories.</p>
<p><a href="http://theagiledeveloper.com/">Matt Deiters</a> implemented some other key user stories, did a lot of user interface implementation work and also wrote a rake task to seed our database with development data.</p>
<p><a href="http://github.com/klobuczek">Heinrich Klobuczek</a> used the <a href="http://github.com/iain/http_accept_language">http_accept_language gem</a> to provide internationalization support, and then proceeded to translate the site into French himself... despite being Polish! We&rsquo;ll have to have Rousseau check his French soon... lol.</p>
<p><img src="http://patshaughnessy.net/assets/2010/10/31/reliefhub4.png"></p>
<p><a href="http://doelsengupta.blogspot.com">Doel Sengupta</a> helped us by testing the app on Windows and posted a couple of tips for getting <a href="http://doelsengupta.blogspot.com/2010/10/uninitialized-constant-win32-nameerror.html">Cucumber</a> and <a href="http://doelsengupta.blogspot.com/2010/10/uninitialized-constant.html">Rails 3</a> working on Windows 7.</p>
<p>Finally, <a href="http://www.futurefridays.com/">Jordan Johnson</a> and I spent most of the week working with the <a href="http://aws.amazon.com/fps/">Amazon Flexible Payment Service</a>; Amazon FPS allows you to collect payments without storing credit card information yourself. They also have a nice option that allows non-profits to avoid transaction fees. Right now we are using the <a href="http://github.com/tylerhunt/remit">Remit gem</a>; I&rsquo;m still looking at some other gems and plugins out there for Amazon FPS/Rails integration.</p>
<p>All in all it was a fantastic, productive week - a great collaboration among people who came together from very different places and companies. While we weren&rsquo;t able to start and finish the entire site in one week, we did get the project off to a good start, and I&rsquo;m sure ReliefHub will be live online fairly soon.</p>

