title: Writing your first PHPUnit test in Drupal
date: 2008/12/12
tag: Drupal

<p>In <a href="http://patshaughnessy.net/2008/12/9/example-drupal-module-to-use-for-tdd-demonstration">my previous post</a> I wrote some typical Drupal code that lists nodes containing a given word in their title. In some upcoming posts I&rsquo;ll rewrite that module using Test Driven Development and see whether the module turns out differently. But to get started with TDD in Drupal we need to be able to write our first test. Before we can even do that we need to install <a href="http://www.phpunit.de">PHPUnit</a>. Once you have PHPUnit installed, test that it is working properly by writing a trivial test file called <a href="http://patshaughnessy.net/assets/code/drupal-tdd-1/FirstTest.php.txt">FirstTest.php</a> somewhere on your hard drive:</p>
<pre>&lt;?php
class FirstTest extends PHPUnit_Framework_TestCase
{
  public function test_two_plus_two_is_four()
  {
    $this-&gt;assertEquals(2+2, 4);
  }
}
?&gt;</pre>
<p>To run this you should use PHPUnit as follows from the same folder containing the test file:</p>
<pre>$ phpunit FirstTest
PHPUnit 3.2.21 by Sebastian Bergmann.
.
Time: 0 seconds
OK (1 test)</pre>
<p>If you get errors running this command then review the <a href="http://www.phpunit.de/manual/3.3/en/installation.html">install instructions for PHPUnit</a> and install it again if necessary.</p>
<p>Next: how do we apply PHPUnit to Drupal? Let&rsquo;s create a new module folder, and an &ldquo;info&rdquo; file in it so Drupal knows what our new module is called: &ldquo;tdd&rdquo; for example. So we create a folder drupal/modules/tdd and a <a href="http://patshaughnessy.net/assets/code/drupal-tdd-1/tdd.info">tdd.info</a> file in this folder that contains:</p>
<pre>name = tdd
description = Module written with TDD
package = TDD Demo Modules
version = VERSION
core = 6.x</pre>
<p>Now let&rsquo;s write our first test. PHPUnit convention is that if you have a PHP class called XYZ, you would write a test class for it called XYZTests, and by default place the test class in XYZTests.php in the same folder. Since we&rsquo;re using Drupal and don&rsquo;t have any object oriented code, let&rsquo;s just call our test class TddTests, named after our new module, and put it into a new file in the same folder called <a href="http://patshaughnessy.net/assets/code/drupal-tdd-1/TddTests.php.1.txt">TddTests.php</a>. It needs to be a subclass of PHPUnit_Framework_TestCase like this:</p>
<pre>&lt;?php
class TddTests extends PHPUnit_Framework_TestCase
{
  public function test_tdd_help()
  {
    $this-&gt;assertEquals(
      tdd_help('admin/content/tdd'), "&lt;p&gt;Help for TDD module.&lt;/p&gt;");
  }
}
?&gt;</pre>
<p>This test will call our new module&rsquo;s &ldquo;help&rdquo; function with the path to the module&rsquo;s new page and make sure we get the proper help. This seems like a good first test to write, since it&rsquo;s very simple but still proves that we can call Drupal code from PHPUnit. Let&rsquo;s run it and see what happens&hellip; But first you have to cd to the folder containing the TddTests.php file. PHPUnit also assumes that by default you&rsquo;re executing the tests from the folder containing the test file. Let's try running our test:</p>
<pre>$ cd modules/tdd
$ phpunit TddTests.php 
PHPUnit 3.2.21 by Sebastian Bergmann.
Fatal error: Call to undefined function tdd_help() in
/Users/pat/htdocs/drupal/modules/tdd/TddTests.php on line 6</pre>
<p>Obviously the test fails because we haven&rsquo;t written our new module yet. This seems like a waste of time, but we've taken the first step in the TDD cycle: write a failing unit test. Now let's start writing <a href="http://patshaughnessy.net/assets/code/drupal-tdd-1/tdd.module">tdd.module</a>:</p>
<pre>&lt;?php
function tdd_help($path, $arg) {
  switch ($path) {
    case &#x27;admin/content/tdd&#x27;:  
      return '&lt;p&gt;Help for TDD module.&lt;/p&gt;';
  }
}
?&gt;</pre>
<p>If you run this test again you&rsquo;ll get the same error message. So now what&rsquo;s wrong? We forgot to enable our new module in the Drupal admin console, so the &ldquo;tdd_help&rdquo; function is still undefined. To enable it, open the Drupal admin console, go to the Administer-&gt;Site Building-&gt;Modules page and look at the bottom for a section called &ldquo;TDD Demo Modules.&rdquo;</p>
<p>But now if you run PHPUnit you&rsquo;ll <i>still</i> get the same error. The mostly empty &ldquo;TDD&rdquo; module is now setup and working inside of Drupal, but PHPUnit has no idea what Drupal is or how to load it. What we need to do is declare somehow in TddTests.php to include and initialize the Drupal framework. To get this to work, we can use the index.php file in the root folder of Drupal app as an example. This is the PHP file that handles all incoming requests to Drupal by initializing the framework, executing the proper menu callback and displaying the result. If you look at the top of the index.php file, you&rsquo;ll see these two lines:</p>
<pre>require_once &#x27;./includes/bootstrap.inc&#x27;;
drupal_bootstrap(DRUPAL_BOOTSTRAP_FULL);</pre>
<p>If we copy these 2 lines into <a href="http://patshaughnessy.net/assets/code/drupal-tdd-1/TddTests.php.2.txt">TddTests.php</a> like this:</p>
<pre>&lt;?php
require_once &#x27;./includes/bootstrap.inc&#x27;;
drupal_bootstrap(DRUPAL_BOOTSTRAP_FULL);
class TddTests extends PHPUnit_Framework_TestCase
{
  public function test_tdd_help()
  {
    $this-&gt;assertEquals(
      tdd_help(&#x27;admin/content/tdd&#x27;), &quot;&lt;p&gt;Help for TDD module.&lt;/p&gt;&quot;);
  }
}
?&gt;</pre>
<p>&hellip; and try to run our test again we get:</p>
<pre>$ phpunit TddTests
Warning: require_once(./includes/bootstrap.inc): failed to open stream:
No such file or directory in /Users/pat/htdocs/drupal/modules/tdd/TddTests.php on line 2
Fatal error: require_once(): Failed opening required
&#x27;./includes/bootstrap.inc&#x27;
(include_path=&#x27;.:/usr/local/php5x/lib/php&#x27;) in
/Users/pat/htdocs/drupal/modules/tdd/TddTests.php on line 2</pre>
<p>The problem now is that the include line has the wrong relative path. If you corrected it you would still get more errors trying to include other Drupal files. It turns out that the drupal_bootstrap function assumes that the current directory is set to the root folder of your app: the location of index.php. To get it all to work, we just need to execute the tests from the root folder:</p>
<pre>$ cd ~/htdocs/drupal
$ phpunit modules/tdd/TddTests.php 
PHPUnit 3.2.21 by Sebastian Bergmann.
Class modules/tdd/TddTests could not be found in modules/tdd/TddTests.php.</pre>
<p>This last error appears because we aren&rsquo;t using the PHPUnit command line properly. The first parameter is the test class name; the second optional parameter that we need to use now is the test file path. Here&rsquo;s the proper command line to use:</p>
<pre>$ phpunit TddTests modules/tdd/TddTests.php 
PHPUnit 3.2.21 by Sebastian Bergmann.
.
Time: 0 seconds
OK (1 test)</pre>
<p>Finally we&rsquo;ve successfully written and executed our first test! More than that, this is our first step toward writing a Drupal module using TDD. Next time I&rsquo;ll get to work writing the rest of the module using TDD with these files as a starting point.</p>
