title: Using LEFT OUTER JOIN with SearchLogic
date: 2010/07/09
tag: Ruby

<p>LEFT OUTER JOIN queries are a great way to handle associations that may contain missing or null values. For example, suppose I have an application with a has_one/belongs_to association between books and authors.</p>
<div class="CodeRay">
<div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>
<span class="r">end</span>
<span class="r">class</span> <span class="cl">Author</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  belongs_to <span class="sy">:book</span>
<span class="r">end</span></pre></div>
</div><br>
<p>If I want to identify the books that don&rsquo;t have any authors, I can use this SQL statement:</p>
<div class="CodeRay">
  <div class="code"><pre>SELECT * FROM books LEFT OUTER JOIN authors ON authors.book_id = books.id
WHERE authors.book_id IS NULL</pre></div>
</div><br>
<p>I need to use LEFT OUTER JOIN here since a normal INNER JOIN query would only return books that have authors. Or if I want to sort the books by their author, I could use:</p>
<div class="CodeRay">
  <div class="code"><pre>SELECT * FROM books LEFT OUTER JOIN  authors ON authors.book_id = books.id
ORDER BY authors.name</pre></div>
</div><br>
<p>This sort would work even if some of the books didn&rsquo;t have an author assigned to them. If we had used a normal INNER JOIN query the books with no author would have been dropped from the sorted list.</p>
<p>Today I&rsquo;m going to discuss how to use LEFT OUTER JOIN with ActiveRecord and SearchLogic, allowing you to handle associations that might have missing records easily and cleanly.</p>
<p><b>ActiveRecord :joins</b></p>
<p>Before we go any farther, let&rsquo;s setup the book/author example so we can explore how ActiveRecord handles joins:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails outer_join_example
$ cd outer_join_example
$ script/generate model book title:string
$ script/generate model author name:string book_id:integer
$ rake db:migrate</pre></div>
</div><br>
<p>And don&rsquo;t forget to add the has_one/belongs_to lines to the two new models:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>
<span class="r">end</span>
<span class="r">class</span> <span class="cl">Author</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  belongs_to <span class="sy">:book</span>
<span class="r">end</span></pre></div>
</div><br>
<p>And now let&rsquo;s create some books and authors we can use to test with:</p>
<div class="CodeRay">
  <div class="code"><pre>$ script/console 
Loading development environment (Rails 2.3.5)
&gt;&gt; [ 'One', 'Two', 'Three' ].each do |title|
?&gt; book = Book.create :title =&gt; title
&gt;&gt; book.author = Author.create :name =&gt; &quot;Author of Book #{title}&quot;
&gt;&gt; book.save
&gt;&gt; end</pre></div>
</div><br>
<p>Let&rsquo;s get started by looking at how you would sort the books by their author&rsquo;s name, using a normal INNER JOIN query:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; ActiveRecord::Base.logger = Logger.new(STDOUT)
&gt;&gt; Book.find(:all, :joins =&gt; :author, :order =&gt; 'authors.name')
       .collect { |book| book.title }
   Book Load (1.4ms)   SELECT &quot;books&quot;.* FROM &quot;books&quot; INNER JOIN &quot;authors&quot;
                       ON authors.book_id = books.id ORDER BY name
=&gt; [&quot;One&quot;, &quot;Three&quot;, &quot;Two&quot;]</pre></div>
</div><br>
<p>Great &ndash; here using find with the :joins option we&rsquo;ve told ActiveRecord to load all the books, and to join with the authors table so we can sort on the authors.name column. But watch what happens if I add a couple of new books that have no author yet, and then repeat the same INNER JOIN sort query:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.create :title =&gt; 'Four'
&gt;&gt; Book.create :title =&gt; 'Five'
&gt;&gt; Book.find(:all, :joins =&gt; :author, :order =&gt; 'name')
       .collect { |book| book.title }
   Book Load (1.8ms)   SELECT &quot;books&quot;.* FROM &quot;books&quot; INNER JOIN &quot;authors&quot;
                       ON authors.book_id = books.id ORDER BY name
=&gt; [&quot;One&quot;, &quot;Three&quot;, &quot;Two&quot;]</pre></div>
</div><br>
<p>Books &ldquo;Four&rdquo; and &ldquo;Five&rdquo; are dropped entirely. This is because INNER JOIN only includes records in the result set that have values from both the books and authors tables. Since there is no author record for these books, they don&rsquo;t appear at all.</p>
<p><b>ActiveRecord :include</b></p>
<p>The simplest way to get around this problem is to use ActiveRecord&rsquo;s :include option, instead of the :joins option. Let&rsquo;s rewrite that call to Book.find and use :include instead:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.find(:all, :include =&gt; :author, :order =&gt; 'authors.name')
       .collect { |book| book.title }
   Book Load Including Associations (2.8ms)   SELECT &quot;books&quot;.&quot;id&quot; AS t0_r0,
&quot;books&quot;.&quot;title&quot; AS t0_r1, &quot;books&quot;.&quot;created_at&quot; AS t0_r2,
&quot;books&quot;.&quot;updated_at&quot; AS t0_r3, &quot;authors&quot;.&quot;id&quot; AS t1_r0,
&quot;authors&quot;.&quot;name&quot; AS t1_r1, &quot;authors&quot;.&quot;book_id&quot; AS t1_r2,
&quot;authors&quot;.&quot;created_at&quot; AS t1_r3, &quot;authors&quot;.&quot;updated_at&quot; AS t1_r4 FROM &quot;books&quot;
LEFT OUTER JOIN &quot;authors&quot; ON authors.book_id = books.id ORDER BY authors.name
=&gt; [&quot;Four&quot;, &quot;Five&quot;, &quot;One&quot;, &quot;Three&quot;, &quot;Two&quot;]</pre></div>
</div><br>
<p>Now we get books &ldquo;Four&rdquo; and &ldquo;Five&rdquo; in the sorted list; this is because ActiveRecord uses a LEFT OUTER JOIN query when you specify the :include option. Note they appear first since their author name values are both null, which is sorted before any of the other authors. If you read the ActiveRecord log output, you&rsquo;ll see it generated a very complex SQL statement that explicitly mentions each column of both the books and authors tables. It used a column naming pattern, &ldquo;t0_r1&rdquo; etc., to identify each column. The reason for all of this is that ActiveRecord is executing the &ldquo;Load Including Associations&rdquo; operation, which is loading each and every attribute for all of the associated objects. This guarantees that we get every possible value for all the books and their authors.</p>
<p>So this is perfect! Now I can write a named_scope to sort books by their author name, including null authors, like this:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>
  named_scope <span class="sy">:sorted_by_author_with_nulls</span>,
    { <span class="sy">:include</span> =&gt; <span class="sy">:author</span>, <span class="sy">:order</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">authors.name</span><span class="dl">'</span></span> }
<span class="r">end</span></pre></div>
</div><br>
<p>And trying it in the console:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.sorted_by_author_with_nulls.collect { |book| book.title }
  Book Load Including Associations (3.9ms)   SELECT &quot;books&quot;.&quot;id&quot; AS t0_r0,
&quot;books&quot;.&quot;title&quot; AS t0_r1, &quot;books&quot;.&quot;created_at&quot; AS t0_r2,
&quot;books&quot;.&quot;updated_at&quot; AS t0_r3, &quot;authors&quot;.&quot;id&quot; AS t1_r0,
&quot;authors&quot;.&quot;name&quot; AS t1_r1, &quot;authors&quot;.&quot;book_id&quot; AS t1_r2,
&quot;authors&quot;.&quot;created_at&quot; AS t1_r3, &quot;authors&quot;.&quot;updated_at&quot; AS t1_r4 FROM &quot;books&quot;
LEFT OUTER JOIN &quot;authors&quot; ON authors.book_id = books.id ORDER BY authors.name
=&gt; [&quot;Four&quot;, &quot;Five&quot;, &quot;One&quot;, &quot;Three&quot;, &quot;Two&quot;]</pre></div>
</div><br>
<p>I can now also write this named_scope called &ldquo;missing_author,&rdquo; which returns just the books that have a missing author:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>
  named_scope <span class="sy">:sorted_by_author_with_nulls</span>,
    { <span class="sy">:include</span> =&gt; <span class="sy">:author</span>, <span class="sy">:order</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">authors.name</span><span class="dl">'</span></span> }
  named_scope <span class="sy">:missing_author</span>,
    { <span class="sy">:include</span> =&gt; <span class="sy">:author</span>, <span class="sy">:conditions</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">authors.name IS NULL</span><span class="dl">'</span></span> }
<span class="r">end</span></pre></div>
</div><br>
<p>And in the console:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.missing_author.collect { |book| book.title }
  Book Load Including Associations (1.9ms)   SELECT &quot;books&quot;.&quot;id&quot; AS t0_r0,
&quot;books&quot;.&quot;title&quot; AS t0_r1, &quot;books&quot;.&quot;created_at&quot; AS t0_r2,
&quot;books&quot;.&quot;updated_at&quot; AS t0_r3, &quot;authors&quot;.&quot;id&quot; AS t1_r0,
&quot;authors&quot;.&quot;name&quot; AS t1_r1, &quot;authors&quot;.&quot;book_id&quot; AS t1_r2,
&quot;authors&quot;.&quot;created_at&quot; AS t1_r3, &quot;authors&quot;.&quot;updated_at&quot; AS t1_r4 FROM &quot;books&quot;
LEFT OUTER JOIN &quot;authors&quot; ON authors.book_id = books.id
WHERE (authors.name IS NULL)
=&gt; [&quot;Four&quot;, &quot;Five&quot;]</pre></div>
</div><br>
<p><b>What’s wrong with :include?</b></p>
<p>It sounds like I&rsquo;m done and that the :include option is the perfect way to handle associations that might contain null or missing values. But not so fast&hellip; it turns out that using :include is often a bad idea. A great resource on :joins and :include options is <a href="http://railscasts.com/episodes/181-include-vs-joins">Ryan Bates&rsquo;s screen cast from 2009</a>. If you think you need to use one of these find options, invest a few minutes and listen to Ryan&rsquo;s explanation. Here I&rsquo;ll just mention a couple of potential problems with using :include with named scopes:</p>
<p>1. It&lsquo;s potentially slow and wasteful. :include causes ActiveRecord to load every attribute for every included associated object, which is often much more information than you really need. If your named scope only requires one or two columns from the associated table, then using :include might be overkill.</p>
<p>2. As Ryan mentions, you lose control over the &ldquo;SELECT&rdquo; portion of the query. This means that if you write a named scope that uses :include, like the missing_author example above, then you can&rsquo;t chain it together with other named scopes that contain a select option. For example, you couldn&rsquo;t write code like this:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="co">Book</span>.missing_author.scoped(<span class="sy">:select</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">DISTINCT title</span><span class="dl">'</span></span>)</pre></div>
</div><br>
<p>This might appear to work, but if you look at the query ActiveRecord generates you&rsquo;ll notice that it doesn&rsquo;t contain the DISTINCT keyword. This is because the Load Including Associations query ignores the select scope.</p>
<p>A better way to write a named_scope that uses LEFT OUTER JOIN is actually to go back to the :joins option, like this:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>
  named_scope <span class="sy">:sorted_by_author_with_nulls</span>, {
    <span class="sy">:joins</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">LEFT OUTER JOIN authors ON authors.book_id = books.id</span><span class="dl">'</span></span>,
    <span class="sy">:order</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">authors.name</span><span class="dl">'</span></span>
  }
  named_scope <span class="sy">:missing_author</span>, {
    <span class="sy">:joins</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">LEFT OUTER JOIN authors ON authors.book_id = books.id</span><span class="dl">'</span></span>,
    <span class="sy">:conditions</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">authors.id IS NULL</span><span class="dl">'</span></span>
  }
<span class="r">end</span></pre></div>
</div><br>
<p>Here I&rsquo;ve written the join clause of the query for each scope, typing in the LEFT OUTER JOIN syntax explicitly. Here&rsquo;s the example from above using DISTINCT:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.missing_author.scoped(:select =&gt; 'DISTINCT title')
   Book Load (1.1ms)   SELECT DISTINCT title FROM &quot;books&quot;
                       LEFT OUTER JOIN authors ON authors.book_id = books.id
                       WHERE (authors.id IS NULL) 
=&gt; [#&lt;Book title: &quot;Four&quot;&gt;, #&lt;Book title: &quot;Five&quot;&gt;]</pre></div>
</div><br>
<p>It works now since the :joins option simply adds LEFT OUTER JOIN to the query without rewriting the entire SELECT statement. Then ActiveRecord combines the join with the select scope, inserting the DISTINCT keyword into the query.</p>
<p><b>Customizing SearchLogic to use LEFT OUTER JOIN</b></p>
<p>Ok &ndash; now using the same techniques from <a href="http://patshaughnessy.net/2010/6/12/using-method_missing-to-customize-searchlogic">my last post</a>, let&rsquo;s see if we can customize SearchLogic to support this sort of named scope. First let&rsquo;s install SearchLogic into our sample app:</p>
<div class="CodeRay">
  <div class="code"><pre>$ script/plugin install http://github.com/binarylogic/searchlogic.git</pre></div>
</div><br>
<p>And next, let&rsquo;s implement a new named scoped called &ldquo;without_author&rdquo; that will work the same way as the &ldquo;missing_author&rdquo; scope from above. We&rsquo;ll use method_missing the same way that SearchLogic does; see my last article for a more complete explanation. Here we go:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>
  <span class="r">class</span> &lt;&lt; <span class="cl">self</span>
    <span class="r">def</span> <span class="fu">method_missing</span>(name, *args, &amp;block)
      <span class="r">if</span> name == <span class="sy">:without_author</span>
        named_scope <span class="sy">:without_author</span>,
          {
            <span class="sy">:joins</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">LEFT OUTER JOIN authors
                       ON authors.book_id = books.id</span><span class="dl">'</span></span>,
            <span class="sy">:conditions</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">authors.id IS NULL</span><span class="dl">'</span></span>
          }
          without_author
      <span class="r">else</span>
        <span class="r">super</span>
      <span class="r">end</span>
    <span class="r">end</span>
  <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br>
<p>And let&rsquo;s try it in the console:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.without_author.collect { |book| book.title }
   Book Load (1.3ms)   SELECT &quot;books&quot;.* FROM &quot;books&quot; LEFT OUTER JOIN authors
   ON authors.book_id = books.id WHERE (authors.id IS NULL) 
=&gt; [&quot;Four&quot;, &quot;Five&quot;]
</pre></div>
</div><br>
<p>Works perfectly! All we need to do now is generalize this for any model and association. Following the pattern from the <a href="http://github.com/binarylogic/searchlogic/blob/master/lib/searchlogic/named_scopes/association_ordering.rb">SearchLogic source code</a>, to get a list of associated models I can call &ldquo;reflect_on_all_associations.&rdquo; This will return a list of AssociationReflection objects, each of which represents an association between this model (Book) and some other model (Author). See the <a href="http://github.com/rails/rails/blob/v2.3.8/activerecord/lib/active_record/reflection.rb#L55">ActiveRecord source code</a> for more details.</p>
<p>Here&rsquo;s what the code looks like with the call to reflect_on_all_associations:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>
  <span class="r">class</span> &lt;&lt; <span class="cl">self</span>
    <span class="r">def</span> <span class="fu">method_missing</span>(name, *args, &amp;block)
      association_names =
        reflect_on_all_associations.collect { |assoc| assoc.name }
      <span class="r">if</span> name.to_s =~ <span class="rx"><span class="dl">/</span><span class="k">^without_(</span></span><span class="il"><span class="idl">#{</span>association_names.join(<span class="s"><span class="dl">&quot;</span><span class="k">|</span><span class="dl">&quot;</span></span>)<span class="idl">}</span></span><span class="rx"><span class="k">)$</span><span class="dl">/</span></span>
        named_scope <span class="sy">:without_author</span>,
          {
            <span class="sy">:joins</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">LEFT OUTER JOIN authors
                       ON authors.book_id = books.id</span><span class="dl">'</span></span>,
            <span class="sy">:conditions</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">authors.id IS NULL</span><span class="dl">'</span></span>
          }
          without_author
      <span class="r">else</span>
        <span class="r">super</span>
      <span class="r">end</span>
    <span class="r">end</span>
  <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br>
<p>You can see that we construct a regex pattern from a list of associated model names, joined with the | character. For example, if there were 3 associated models, then we would use /without_(assoc1|assoc2|assoc3)/&hellip; in this example since Book has only one associated model, we have a simple regex pattern: /without_(author)/.</p>
<p>Let&rsquo;s make sure the code still works in the console:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.without_author.collect { |book| book.title }
   Book Load (1.0ms)   SELECT &quot;books&quot;.* FROM &quot;books&quot; LEFT OUTER JOIN authors
   ON authors.book_id = books.id WHERE (authors.id IS NULL) 
=&gt; [&quot;Four&quot;, &quot;Five&quot;]</pre></div>
</div><br>
<p>So far so good. The next thing we need to do is replace the hard coded values in the call to named_scope. To do this, we&rsquo;ll need the AssociationReflection object that corresponds to the matching association name. Here&rsquo;s some code that does that:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">def</span> <span class="fu">matching_association</span>(match)
  <span class="iv">@matching_association</span> ||= reflect_on_all_associations.detect <span class="r">do</span> |assoc|
    assoc.name.to_s == match
  <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br>
<p>If we pass in the matching name from the regex pattern above, e.g. matching_association($1), then we&rsquo;ll get the corresponding association object. Once we have that, we can get the name of that association&rsquo;s database table and primary key:</p>
<div class="CodeRay">
  <div class="code"><pre>associated_table = matching_association(<span class="gv">$1</span>).table_name
associated_key = matching_association(<span class="gv">$1</span>).primary_key_name</pre></div>
</div><br>
<p>One subtle point to note here: ActiveRecord's AssociationReflection.primary_key_name method actually returns the foreign key column for this association, the book_id column in the authors table, and not the primary key of the authors table, which would have just been id. It works properly, but possibly should have been named foreign_key_name. Anyway, now we&rsquo;ll need these values to construct our LEFT OUTER JOIN and condition strings. These methods do that:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">def</span> <span class="fu">join_clause</span>(associated_table, associated_key)
  outer_join = <span class="s"><span class="dl">&quot;</span><span class="k">LEFT OUTER JOIN </span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
  outer_join += <span class="s"><span class="dl">&quot;</span><span class="k"> ON </span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>associated_key<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
  outer_join += <span class="s"><span class="dl">&quot;</span><span class="k"> = </span></span><span class="il"><span class="idl">#{</span>quoted_table_name<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>primary_key<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
<span class="r">end</span>

<span class="r">def</span> <span class="fu">where_clause</span>(associated_table, associated_key)
  <span class="s"><span class="dl">&quot;</span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>associated_key<span class="idl">}</span></span><span class="s"><span class="k"> IS NULL</span><span class="dl">&quot;</span></span>
<span class="r">end</span></pre></div>
</div><br>
<p>Finally, we can put it all together like this:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Book</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>
  has_one <span class="sy">:author</span>

  <span class="r">class</span> &lt;&lt; <span class="cl">self</span>
    <span class="r">def</span> <span class="fu">method_missing</span>(name, *args, &amp;block)
      association_names =
        reflect_on_all_associations.collect { |assoc| assoc.name }
      <span class="r">if</span> name.to_s =~ <span class="rx"><span class="dl">/</span><span class="k">^without_(</span></span><span class="il"><span class="idl">#{</span>association_names.join(<span class="s"><span class="dl">&quot;</span><span class="k">|</span><span class="dl">&quot;</span></span>)<span class="idl">}</span></span><span class="rx"><span class="k">)$</span><span class="dl">/</span></span>
        associated_table = matching_association(<span class="gv">$1</span>).table_name
        associated_key = matching_association(<span class="gv">$1</span>).primary_key_name
        named_scope name,
          {
            <span class="sy">:joins</span> =&gt; join_clause(associated_table, associated_key),
            <span class="sy">:conditions</span> =&gt; where_clause(associated_table, associated_key)
          }
        send(name)
      <span class="r">else</span>
        <span class="r">super</span>
      <span class="r">end</span>
    <span class="r">end</span>

    <span class="r">def</span> <span class="fu">matching_association</span>(match)
      <span class="iv">@matching_association</span> ||= reflect_on_all_associations.detect <span class="r">do</span> |assoc|
        assoc.name.to_s == match
      <span class="r">end</span>
    <span class="r">end</span>

    <span class="r">def</span> <span class="fu">join_clause</span>(associated_table, associated_key)
      outer_join = <span class="s"><span class="dl">&quot;</span><span class="k">LEFT OUTER JOIN </span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
      outer_join += <span class="s"><span class="dl">&quot;</span><span class="k"> ON </span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>associated_key<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
      outer_join += <span class="s"><span class="dl">&quot;</span><span class="k"> = </span></span><span class="il"><span class="idl">#{</span>quoted_table_name<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>primary_key<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
    <span class="r">end</span>

    <span class="r">def</span> <span class="fu">where_clause</span>(associated_table, associated_key)
      <span class="s"><span class="dl">&quot;</span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>associated_key<span class="idl">}</span></span><span class="s"><span class="k"> IS NULL</span><span class="dl">&quot;</span></span>
    <span class="r">end</span>
  <span class="r">end</span>
<span class="r">end</span></pre></div>
</div><br>
<p>Note that I used &ldquo;send(name)&rdquo; to call the named scope we just created with named_scope. Let&rsquo;s make sure it still all works properly in the console:</p>
<div class="CodeRay">
  <div class="code"><pre>&gt;&gt; Book.without_author.collect { |book| book.title }
   Book Load (1.0ms)   SELECT &quot;books&quot;.* FROM &quot;books&quot;
   LEFT OUTER JOIN authors ON authors.book_id = &quot;books&quot;.id
   WHERE (authors.book_id IS NULL) 
=&gt; [&quot;Four&quot;, &quot;Five&quot;]</pre></div>
</div><br>
<p>Phew &ndash; it does! Ideally I would have some RSpec examples setup to test this, instead of using the console.</p>
<p>Just like in <a href="http://patshaughnessy.net/2010/6/12/using-method_missing-to-customize-searchlogic">my last article</a>, the last thing I&rsquo;ll do today is move these class methods out of the Book model and into a new module called SearchLogicExtensions, which in my application I saved into a file called config/initializers/search_logic_extensions.rb. This causes the method missing code to be loaded when my app starts up. At the bottom I extend ActiveRecord::Base with the new module, so the named scope can be used by every model in my application:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">module</span> <span class="cl">SearchLogicExtensions</span>
  <span class="r">def</span> <span class="fu">method_missing</span>(name, *args, &amp;block)
    association_names =
      reflect_on_all_associations.collect { |assoc| assoc.name }
    <span class="r">if</span> name.to_s =~ <span class="rx"><span class="dl">/</span><span class="k">^without_(</span></span><span class="il"><span class="idl">#{</span>association_names.join(<span class="s"><span class="dl">&quot;</span><span class="k">|</span><span class="dl">&quot;</span></span>)<span class="idl">}</span></span><span class="rx"><span class="k">)$</span><span class="dl">/</span></span>
      associated_table = matching_association(<span class="gv">$1</span>).table_name
      associated_key = matching_association(<span class="gv">$1</span>).primary_key_name
      named_scope name,
        {
          <span class="sy">:joins</span> =&gt; join_clause(associated_table, associated_key),
          <span class="sy">:conditions</span> =&gt; where_clause(associated_table, associated_key)
        }
      send(name)
    <span class="r">else</span>
      <span class="r">super</span>
    <span class="r">end</span>
  <span class="r">end</span>

  <span class="r">def</span> <span class="fu">matching_association</span>(match)
    <span class="iv">@matching_association</span> ||= reflect_on_all_associations.detect <span class="r">do</span> |assoc|
        assoc.name.to_s == match
    <span class="r">end</span>
  <span class="r">end</span>

  <span class="r">def</span> <span class="fu">join_clause</span>(associated_table, associated_key)
    outer_join = <span class="s"><span class="dl">&quot;</span><span class="k">LEFT OUTER JOIN </span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
    outer_join += <span class="s"><span class="dl">&quot;</span><span class="k"> ON </span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>associated_key<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
    outer_join += <span class="s"><span class="dl">&quot;</span><span class="k"> = </span></span><span class="il"><span class="idl">#{</span>quoted_table_name<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>primary_key<span class="idl">}</span></span><span class="s"><span class="dl">&quot;</span></span>
  <span class="r">end</span>

  <span class="r">def</span> <span class="fu">where_clause</span>(associated_table, associated_key)
    <span class="s"><span class="dl">&quot;</span></span><span class="il"><span class="idl">#{</span>associated_table<span class="idl">}</span></span><span class="s"><span class="k">.</span></span><span class="il"><span class="idl">#{</span>associated_key<span class="idl">}</span></span><span class="s"><span class="k"> IS NULL</span><span class="dl">&quot;</span></span>
  <span class="r">end</span>
<span class="r">end</span>

<span class="co">ActiveRecord</span>::<span class="co">Base</span>.extend(<span class="co">SearchLogicExtensions</span>)</pre></div>
</div>
