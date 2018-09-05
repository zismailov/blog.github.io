title: Using MySQL transactions with Drupal unit tests
date: 2009/01/19
tag: Drupal

<p><a href="http://patshaughnessy.net/2009/1/16/using-a-test-database-with-drupal-unit-tests">Last time</a> I wrote about how to use an entirely separate MySQL database to hold test data for Drupal unit tests, similar to the approach that the Ruby on Rails framework uses for test data. In this post I&rsquo;ll look at another Rails innovation that can be applied equally well to unit testing with Drupal: running each unit test in a separate database transaction.</p>
<p>First let&rsquo;s take a quick look at what database transactions are, and how we would use them while running unit tests. In a nutshell, a database transaction is just a way to group a series of SQL operations together and insuring that they are all run together as a single unit &ndash; either all of them are executed, or none of them. Let&rsquo;s take an example. Here are some of the SQL statements Drupal executes when you save a new node in the database:</p>
<pre>BEGIN
INSERT INTO node_revisions (nid, uid, title, body, teaser, log, timestamp...
INSERT INTO node (vid, type, language, title, uid, status, created, changed...
UPDATE node_revisions SET nid = 2 WHERE vid = 2
COMMIT</pre>
<p>Normally Drupal does not use transactions, but I've inserted the &ldquo;BEGIN&rdquo; and &ldquo;COMMIT&rdquo; commands here as an example: the transaction starts with the BEGIN command, and ends with the COMMIT command. When MySQL receives the COMMIT command, it allows other database clients (future Drupal HTTP requests, or possibly future command line unit tests) to see the new inserted and updated node data. However, if the transaction were rolled back like this:</p>
<pre>BEGIN
INSERT INTO node_revisions (nid, uid, title, body, teaser, log, timestamp...
INSERT INTO node (vid, type, language, title, uid, status, created, changed...
UPDATE node_revisions SET nid = 2 WHERE vid = 2
ROLLBACK</pre>
<p>&hellip; then none of the changes would be made to the node and node_revisions tables. Instead when MySQL receives the ROLLBACK command it will discard the changes and these tables will appear the same way they did before the transaction started. Therefore, by running each unit test in a separate transaction and rolling it back at the end of each test, we can insure that any changes made to the database by that test are discarded&hellip; before the next test is run. Below I&rsquo;ll explain how to actually do this with PHPUnit and Drupal.</p>
<p>But first let me quickly mention another huge benefit to using transactions with unit test suites: test performance. To learn more about why your tests will run a lot faster using MySQL transactions, read <a href="http://clarkware.com/cgi/blosxom/2005/10/24#Rails10FastTesting">this great article by Mike Clark</a> from the period when Rails 1.0 was released, way back in 2005. What Mike wrote about Rails in 2005 is still true today for Drupal: your unit tests will run faster because fewer SQL statements are required. You won’t need to execute DELETE SQL statements to remove the data after each test since rolling back each transaction accomplishes the same thing.</p>
<p>Now let&rsquo;s get it to work with Drupal&hellip; first let me add a second unit test to my simple PHPUnit test class from <a href="http://patshaughnessy.net/2009/1/16/using-a-test-database-with-drupal-unit-tests">last time</a>:</p>
<pre>&lt;?php
require_once &#x27;./includes/phpunit_setup.inc&#x27;;
class TestDataExampleTest2 extends PHPUnit_Framework_TestCase
{  
  public function create_test_blog_post()
  {
    $node = new stdClass();
    $node-&gt;title = &quot;This is a blog post&quot;;
    $node-&gt;body = &quot;This is the body of the post&quot;;
    $node-&gt;type = &quot;Story&quot;;
    $node-&gt;promote = 1;
    node_save($node);
    return $node;
  }
  public function test_there_is_one_post()
  {
    $this-&gt;create_test_blog_post();
    $this-&gt;assertEquals(1, db_result(db_query(&quot;SELECT COUNT(*) FROM {NODE}&quot;)));
  }
  public function test_there_are_two_posts()
  {
    $this-&gt;create_test_blog_post();
    $this-&gt;create_test_blog_post();
    $this-&gt;assertEquals(2, db_result(db_query(&quot;SELECT COUNT(*) FROM {NODE}&quot;)));
  }
}
?&gt;</pre>
<p>In this example, I use the <a href="http://patshaughnessy.net/assets/code/drupal-tdd-4/phpunit_setup.inc">phpunit_setup.inc</a> file I wrote in my last post to clear out and setup a new Drupal schema each time I run the tests. Even though I have a clean test database each time I run PHPUnit, without using transactions one of these two unit tests will fail since each one creates its own test data, and assumes no other test data exist in the node table:</p>
<pre>$ phpunit TestDataExampleTest2
          modules/test_data_module/TestDataExampleTest2.php 
PHPUnit 3.2.21 by Sebastian Bergmann.
.F
Time: 0 seconds
There was 1 failure:
1) test_there_are_two_posts(TestDataExampleTest2)
Failed asserting that &lt;string:3&gt; matches expected value &lt;integer:2&gt;.
/Users/pat/htdocs/drupal4/modules/test_data_module/TestDataExampleTest2.php:24
FAILURES!
Tests: 2, Failures: 1.</pre>
<p>Here the second test fails since the blog post created in the first test is still present in the database. The simplest way to start a new database transaction before each test is run, and to rollback after each test is completed, is with the PHPUnit setup/teardown methods as follows:</p>
<pre>public function setup()
{
  db_query(&quot;BEGIN&quot;);
}
public function teardown()
{
  db_query(&quot;ROLLBACK&quot;);
}</pre>
<p>If you add these functions to the &ldquo;TestDataExampleTest2&rdquo; class above both tests should now pass since the ROLLBACK call will delete the nodes created by each test each time teardown is called&hellip;</p>
<pre>$ phpunit TestDataExampleTest2
          modules/test_data_module/TestDataExampleTest2.php 
PHPUnit 3.2.21 by Sebastian Bergmann.
.F
&hellip;
FAILURES!
Tests: 2, Failures: 1.</pre>
<p>Wait&hellip; what happened? It failed!</p>
<p>The problem is that MySQL does not support transactions using the MyISAM database engine, which is what Drupal uses by default. What we need to do is to convert all of the Drupal MySQL tables to use the InnoDB database engine instead. Unfortunately, there are many implications to using InnoDB vs. MyISAM in Drupal or with any MySQL based application. See "<a href="http://2bits.com/articles/mysql-innodb-performance-gains-as-well-as-some-pitfalls.html">MySQL InnoDB: performance gains as well as some pitfalls</a>" to read more. Specifically, there can be performance issues and degradation when using InnoDB incorrectly, or depending on the type of application you have. Drupal was actually designed and developed with MyISAM in mind, and not InnoDB, although there is some chance this might change for Drupal 7 someday.</p>
<p>Despite all of this, using InnoDB in a <b>test database</b> is a great idea since you will get all of the benefits of isolating tests from each other without having to worry about how InnoDB will effect your production site&rsquo;s performance. In fact, the performance of your tests will actually be dramatically improved, <a href="http://clarkware.com/cgi/blosxom/2005/10/24#Rails10FastTesting">as Mike Clark explained</a>.</p>
<p>With all of this in mind, I wrote some code to convert the newly created Drupal tables in the test database from MyISAM to InnoDB right after we clear out and reload the test database. Here&rsquo;s how it works; this code is from <a href="http://patshaughnessy.net/assets/code/drupal-tdd-4/phpunit_setup.inc">phpunit_setup.inc</a>, which I included at the top of my PHPUnit test file:</p>
<pre>function enable_mysql_transactions()
{
  convert_test_tables_to_innodb();
  db_query(&quot;SET AUTOCOMMIT = 0&quot;);  
}
function convert_test_tables_to_innodb()
{
  each_table(&#x27;convert_to_innodb&#x27;);  
} 
function each_table($table_callback)
{
  global $db_url;
  $url = parse_url($db_url[&#x27;test&#x27;]);
  $database = substr($url[&#x27;path&#x27;], 1);
  $result = db_query(&quot;SELECT table_name FROM information_schema.tables
                      WHERE table_schema = &#x27;$database&#x27;&quot;);
  while ($table = db_result($result)) {
    $table_callback($table);
  }
}
function convert_to_innodb($table)
{
  db_query(&quot;ALTER TABLE $table ENGINE = INNODB&quot;);
}</pre>
<p>This iterates over the Drupal tables in the test database and executes ALTER TABLE &hellip; ENGINE = INNODB on each one. The SET AUTOCOMMIT=0 command is used to prevent SQL statements from being committed immediately after they are executed, and to allow the InnoDB transactions to work properly.</p>
<p>To repeat and summarize how to employ and separate MySQL test database and transactions in your PHPUnit tests for Drupal, just follow these steps:</p>
<ol>
  <li><p>Edit settings.php and use an array of two values for $db_url:</p>
    <pre>$db_url["default"] = 'mysql://user:password@localhost/drupal;
$db_url["test"] = 'mysql://user:password@localhost/drupal_test';</pre></li>
  <li><p>Create a new test database in MySQL:</p><pre>CREATE DATABASE drupal_test DEFAULT CHARACTER SET utf8
                            COLLATE utf8_unicode_ci;</pre></li>
  <li>Download and save <a href="http://patshaughnessy.net/assets/code/drupal-tdd-4/phpunit_setup.inc">phpunit_setup.inc</a> somewhere in your Drupal application; for example in the &ldquo;includes&rdquo; folder.</li>
  <li>Include <a href="http://patshaughnessy.net/assets/code/drupal-tdd-4/phpunit_setup.inc">phpunit_setup.inc</a> at the top of each of your PHPUnit test classes.</li>
  <li><p>Execute your PHPUnit test class from the root folder of your Drupal app:</p><pre>$ cd /path/to/your/drupal-site
$ phpunit YourClass modules/your_module/YourClassFileName.php 
PHPUnit 3.2.21 by Sebastian Bergmann.
..
Time: 0 seconds
OK (2 tests)</pre></li>
</ol>
<p>Here’s my finished test class:</p>
<pre>&lt;?php
require_once &#x27;./includes/phpunit_setup.inc&#x27;;
class TestDataExampleTest2 extends PHPUnit_Framework_TestCase
{  
  public function setup()
  {
    db_query(&quot;BEGIN&quot;);
  }
  public function teardown()
  {
    db_query(&quot;ROLLBACK&quot;);
  }
  public function create_test_blog_post()
  {
    $node = new stdClass();
    $node-&gt;title = &quot;This is a blog post&quot;;
    $node-&gt;body = &quot;This is the body of the post&quot;;
    $node-&gt;type = &quot;Story&quot;;
    $node-&gt;promote = 1;
    node_save($node);
    return $node;
  }
  public function test_there_is_one_post()
  {
    $this-&gt;create_test_blog_post();
    $this-&gt;assertEquals(1, db_result(db_query(&quot;SELECT COUNT(*) FROM {NODE}&quot;)));
  }
  public function test_there_are_two_posts()
  {
    $this-&gt;create_test_blog_post();
    $this-&gt;create_test_blog_post();
    $this-&gt;assertEquals(2, db_result(db_query(&quot;SELECT COUNT(*) FROM {NODE}&quot;)));
  }
}
?&gt;</pre>
