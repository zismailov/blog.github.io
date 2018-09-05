title: Using TDD to write a Drupal module
date: 2008/12/19
tag: Drupal

<p>Last time <a href="http://patshaughnessy.net/2008/12/12/writing-your-first-phpunit-test-in-drupal">I wrote my first PHPUnit test for Drupal</a>. Now I&rsquo;d like to continue writing my new Drupal module using Test Driven Development (TDD). Here are links to the finished test code and production code files that I&rsquo;ll write below: <a href="http://patshaughnessy.net/assets/code/drupal-tdd-2/TddTests.php.txt">TddTests.php</a> and <a href="http://patshaughnessy.net/assets/code/drupal-tdd-2/tdd.module">tdd.module</a>, in case you want to download and try these tests yourself.</p>
<p>With TDD the first step is always to write a failing unit test. But, which test to write first? What should I try to test? Since the demonstration module I&rsquo;m writing will display a series of nodes on the screen containing a certain word in their title, let&rsquo;s try to test that behavior directly, right away:</p>
<pre>public function test_search_for_titles()
{
  $query = &#x27;FindMe&#x27;;
  $titles = tdd_search_for_titles($query);
}</pre>
<p>This doesn&rsquo;t actually test anything, but it calls a function that will return the titles. Now if we run this test, it will obviously fail since &ldquo;tdd_search_for_titles&rdquo; is not defined. Let&rsquo;s write that. Another rule of TDD is to write just enough production code to get the failing unit test to pass. The simplest way to get this test to pass is to just return a hard coded title like this:</p>
<pre>function tdd_search_for_titles($query) {
  return array(&#x27;Hard coded title with the word FindMe&#x27;);
}</pre>
<p>Now the test above passes, along with the first test I wrote in <a href="http://patshaughnessy.net/2008/12/12/writing-your-first-phpunit-test-in-drupal">my last post</a>:</p>
<pre>$ phpunit TddTests modules/tdd/TddTests.php 
PHPUnit 3.2.21 by Sebastian Bergmann.
..
Time: 0 seconds
OK (2 tests)</pre>
<p>This seems silly, but at least we&rsquo;ve started writing and executing code in our new module. Since our test is not actually testing anything, let&rsquo;s add some real test code to it by checking that the titles returned actually contain the query string:</p>
<pre>public function test_search_for_titles()
{
  $query = &#x27;FindMe&#x27;;
  $titles = tdd_search_for_titles($query);
  foreach ($titles as $title) {
    $this-&gt;assertTrue(stripos($title, $query) &gt; 0);
  }
}</pre>
<p>Run the test again: it still passes. It&rsquo;s important when using TDD to continuously run your tests as you write code, as often as every 30 seconds or 1-2 minutes. That way as soon as your tests fail, you know immediately what caused the problem: whatever code you changed during the past 1-2 minutes. One benefit of TDD is that you rarely need to use a debugger to figure out what is wrong since you execute and test your changes so frequently. Next, as a sanity check, let&rsquo;s try breaking the code on purpose and checking if our test is really working or not:</p>
<pre>function tdd_search_for_titles($query) {
  return array(&#x27;Hard coded title with the word FindXYZMe&#x27;);
}</pre>
<p>Now the test fails (good!):</p>
<pre>$ phpunit TddTests modules/tdd/TddTests.php
PHPUnit 3.2.21 by Sebastian Bergmann.
.F
Time: 0 seconds
There was 1 failure:
1) test_search_for_titles(TddTests)
Failed asserting that &lt;boolean:false&gt; is true.
/Users/pat/htdocs/drupal3/modules/tdd/TddTests.php:15
FAILURES!
Tests: 2, Failures: 1.</pre>
<p>I frequently do something like this when some new tests pass for the first time, just to be sure they are really executing and calling the code I intended them to. Sometimes trusting your tests too much can be dangerous! If we remove the &ldquo;XYZ&rdquo; change the test will pass again. Once the test is passing again we can move on.</p>
<p>So far we haven&rsquo;t done much. All we have done is to prove that we can write a function to return a hard-coded string. Let&rsquo;s continue by testing that we can query actual data in the MySQL database. But first we need to create some data for our function to look for. The simplest way to do that would be to open Drupal in a web browser and to create some nodes (web pages) that have the query string in their title. This is a good way to get started, but can lead to trouble down the road since our test will now rely on someone having created these pages manually. If we run the tests using a different database or on someone else&rsquo;s machine they will fail. A better approach would be to have the test itself create the data it expects. Here&rsquo;s how to do that:</p>
<pre>public function test_search_for_titles()
{
  $login_form = array(&#x27;name&#x27; =&gt; &#x27;admin&#x27;, &#x27;pass&#x27; =&gt; &#x27;adminpassword&#x27;);
  user_authenticate($login_form);
  
  $node = new stdClass();
  $node-&gt;title = &#x27;This title contains the word FindSomethingElse&#x27;;
  $node-&gt;body = &#x27;This is the body of the node&#x27;;
  node_save($node);

  $query = &#x27;FindSomethingElse&#x27;;
  $titles = tdd_search_for_titles($query);
  foreach ($titles as $title) {
    $this-&gt;assertTrue(stripos($title, $query) &gt; 0);
  }
  
  node_delete($node-&gt;nid);    
}</pre>
<p>There are a few different things going on here:
  <ul>
    <li>Most importantly we have created a node object in memory, loaded it with some test values, and then saved it into the database using node_save(). This means that every time the test is run, it will create the node that the test code expects to find.</li>
    <li>The call to user_authenticate() at the top is required to setup Drupal properly, allowing the node_save() and node_delete() functions to work. Without it, we would probably get an access denied error since anonymous users typically aren’t allow to create and save pages, and certainly not to delete pages.</li>
    <li>Finally, at the bottom of the test we are calling node_delete(), and passing the node id of the node we created earlier. This is how the test cleans up after itself. Without this our Drupal database would begin to fill up with test records, one new record each time we run the test.</li>
  </ul>
</p>
<p>If we run the test it will fail, of course, since our hard coded title does not contain &ldquo;FindSomethingElse.&rdquo; Instead of changing the hard coded string to make the test pass, let&rsquo;s actually do some real coding and write a SQL statement to get the titles from MySQL:</p>
<pre>function tdd_search_for_titles($query) {
  $titles = array();
  $result = db_query(&quot;SELECT title FROM {node}&quot;);
  while ($node = db_fetch_object($result)) {
    $titles[] = $node-&gt;title;
  }
  return $titles;
}</pre>
<p>Finally we have started writing code that our module will actually use. Here we take the query string, construct a select statement and execute it using db_query(). Once we have the results, we return the titles as an array of strings. Let&rsquo;s run our test and see what happens:</p>
<pre>$ phpunit TddTests modules/tdd/TddTests.php
Failed asserting that &lt;boolean:false&gt; is true.</pre>
<p>Oops &mdash; it failed! What went wrong? If we add another check to the test, we can get some more information:</p>
<pre>$this-&gt;assertEquals(count($titles), 1);</pre>
<p>And if we run again:</p>
<pre>$ phpunit TddTests modules/tdd/TddTests.php
Failed asserting that &lt;integer:1&gt; matches expected value &lt;integer:27&gt;.</pre>
<p>The problem is that there are 27 titles returned, instead of just the one we created. If we take a second look at the SQL statement we see right away that there is no WHERE clause, causing all of the titles in the database to be returned. Now, if we fix the SQL statement, the test should pass:</p>
<pre>$result = db_query(
  &quot;SELECT title FROM {node} WHERE title LIKE &#x27;%%%s%%&#x27;&quot;, $query);</pre>
<p>Let&rsquo;s see:</p>
<pre>phpunit TddTests modules/tdd/TddTests.php
Failed asserting that &lt;integer:1&gt; matches expected value &lt;integer:3&gt;</pre>
<p>Now what&rsquo;s the problem?? It turns out that both the test and production functions are working properly, and there really are 3 nodes in the database with &ldquo;FindSomethingElse&rdquo; in the title. The reason why is that our call to node_delete() was not executed when we ran our test above and it failed (twice). We&rsquo;ll fix that below. For now, let&rsquo;s just clean up the database and remove the extra test records. The best thing to do is just to delete them from the Drupal admin console (Administer-&gt;Content management-&gt;Content). Now when you are sure no other nodes exist with &ldquo;FindSomethingElse&rdquo; in the title run the test again and it will pass.</p>
<p>Now&hellip; why were extra test node records created when the test failed? What happened is that the call to node_delete() is never executed if any of the asserts fail. That is, the test execution stops as soon as there is a failing assert statement. I suppose this makes sense; we know the overall test will fail if there is even one failing assert statement, and so there&rsquo;s no need to continue executing the test code.</p>
<p>The solution is to refactor the test code and use two new functions called setup() and teardown(), as follows:</p>
<pre>public function setup()
  {
    $login_form = array(&#x27;name&#x27; =&gt; &#x27;admin&#x27;, &#x27;pass&#x27; =&gt; &#x27;adminpassword&#x27;);
    user_authenticate($login_form);

    $this-&gt;node = new stdClass();
    $this-&gt;node-&gt;title = &#x27;This title contains the word FindSomethingElse&#x27;;
    $this-&gt;node-&gt;body = &#x27;This is the body of the node&#x27;;
    node_save($this-&gt;node);    
  }
public function teardown()
  {
    node_delete($this-&gt;node-&gt;nid);  
  }</pre>
<p>The way this works is that setup() is called once for each test in the test class, just before the test function is called. In our case it will be called twice: once for test_tdd_help() and once for test_search_for_titles(). This gives us a chance to create test data and perform other setup tasks before each test is executed. As you might guess, teardown() is called once for each test also, right after the test function finishes. This gives us a chances to remove our test data, even if the test fails. One thing to note about this: you have to save the $node object inside the test class so that it can be accessed from teardown() and from the tests themselves if necessary; so &ldquo;$node&rdquo; becomes &ldquo;$this-&gt;node&rdquo;.</p>
<p>Here are the links again to the final test and code files: <a href="http://patshaughnessy.net/assets/code/drupal-tdd-2/TddTests.php.txt">TddTests.php</a> and <a href="http://patshaughnessy.net/assets/code/drupal-tdd-2/tdd.module">tdd.module</a>. In my next post I&rsquo;ll finish up the code for tdd.module and TddTests.php without providing as much detail, and then move quickly to a discussion of how this code can be integrated with Drupal and included in a working web site.</p>
