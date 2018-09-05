title: "How Ruby Borrowed a Decades Old Idea From Lisp"
date: 2012/9/18
tag: Ruby

<b>This is the last of a series of free excerpts from an eBook I’m writing called [Ruby Under a Microscope](http://patshaughnessy.net/ruby-under-a-microscope). I plan to finish the book and make it available for purchase and download from this web site before [RubyConf 2012](http://rubyconf.org) on Nov. 1. You can still [sign up here](http://patshaughnessy.net/ruby-under-a-microscope), if you haven’t already, to receive an email when the book is finished. I plan send that one, single email message out to everyone before November!</b>

<div style="float: left; padding: 17px 30px 10px 0px">
  <table cellpadding="0" cellspacing="0" border="0">
    <tr><td><img src="http://patshaughnessy.net/assets/2012/9/18/ibm-704.jpg"></td></tr>
    <tr><td align="center"><i>The IBM 704, above, was the first computer<br/>to run Lisp in the early 1960s.</i></td></tr>
  </table>
</div>

Blocks are one of the most commonly used and powerful features of Ruby. As you probably know, they allow you to pass a code snippet to iterators such as <span class="code">each</span>, <span class="code">detect</span> or <span class="code">inject</span>. You can also write your own functions that call blocks for other reasons using the <span class="code">yield</span> keyword. Ruby code containing blocks is often more succinct, elegant and expressive than the equivalent code would appear in older languages such as C.

However, don’t jump to the conclusion that blocks are a new idea! In fact, blocks are not new to Ruby at all; the computer science concept behind blocks, called “closures,” [was first invented by Peter J. Landin](http://en.wikipedia.org/wiki/Closure_(computer_science\)#History_and_etymology) in 1964, a few years after the original version of [Lisp](http://en.wikipedia.org/wiki/Lisp_(programming_language\)) was created by [John McCarthy](http://en.wikipedia.org/wiki/John_McCarthy_(computer_scientist\)) in 1958. Closures were later adopted by Lisp - or more precisely a dialect of Lisp called [Scheme](http://en.wikipedia.org/wiki/Scheme_(programming_language\)), invented by [Gerald Sussman](http://en.wikipedia.org/wiki/Gerald_Jay_Sussman) and [Guy Steele](http://en.wikipedia.org/wiki/Guy_L._Steele,_Jr.) in 1975. Sussman and Steele’s use of closures in Scheme brought the idea to many programmers for the first time starting in the 1970s.

But what does “closure” actually mean? In other words, exactly what are Ruby blocks? Are they as simple as they appear? Are they just the snippet of Ruby code that appears between the <span class="code">do</span> and <span class="code">end</span> keywords? Or is there more to Ruby blocks than meets the eye? In this chapter I’ll review how Ruby implements blocks internally, and show how they meet the definition of “closure” used by Sussman and Steele back in 1975. I’ll also show how blocks, lambdas, procs and bindings are all different ways of looking at closures, and how these objects are related to Ruby’s metaprogramming API.

<div style="float: right; padding: 25px 0px 20px 40px">
<table cellpadding="0" cellspacing="0" border="0">
  <tr><td><img src="http://patshaughnessy.net/assets/2012/9/18/sussman-steele-paper.jpg"></td></tr>
  <tr><td align="center"><i>Sussman and Steele gave a useful definition of the term “closure”<br/>in <a href="http://en.wikisource.org/wiki/Scheme:_An_Interpreter_for_Extended_Lambda_Calculus">this 1975 academic paper</a>, one of the so-called “<a href="http://en.wikipedia.org/wiki/Lambda_Papers#The_Lambda_Papers">Lambda Papers</a>.”</i></td></tr>
</table>
</div>

## What exactly is a block?

Internally Ruby represents each block using a C structure called <span class="code">rb_block_t</span>:

<img src="http://patshaughnessy.net/assets/2012/9/18/empty-block.png"/>

One way to define what a block is would be to take a look at the values Ruby stores inside this structure. Just as we did in Chapter 3 with the <span class="code">RClass</span> structure, let’s deduce what the contents of the <span class="code">rb_block_t</span> structure must be based on what we know blocks can do in our Ruby code.

Starting with the most obvious attribute of blocks, we know each block must consist of a piece of Ruby code, or internally a set of compiled YARV byte code instructions. For example, if I call a method and pass a block as a parameter:

<br/>
<pre type="ruby">
10.times do
  str = "The quick brown fox jumps over the lazy dog."
  puts str
end
</pre>

...it’s clear that when executing the <span class="code">10.times</span> call Ruby needs to know what code to iterate over. Therefore, we know the <span class="code">rb_block_t</span> structure must contain a pointer to that code:

<img src="http://patshaughnessy.net/assets/2012/9/18/block-and-code.png"/>

In this diagram you can see a value called <span class="code">iseq</span> which is a pointer to the YARV instructions compiled from the Ruby code in my block.

Another obvious but often overlooked behavior of blocks is that they can access variables in the surrounding or parent Ruby scope. For example:

<br/>
<pre type="ruby">
str = "The quick brown fox"
10.times do
  str2 = "jumps over the lazy dog."
  puts "#{str} #{str2}"
end
</pre>

Here the <span class="code">puts</span> function call refers equally well to the <span class="code">str2</span> variable located inside the block and the <span class="code">str</span> variable from the surrounding code. We often take this for granted - obviously blocks can access values from the code surrounding them. This ability is one of the things that makes blocks useful. But if you think about this for a moment, you’ll realize blocks have in some sense a dual personality. On the one hand they behave like separate functions: you can call them and pass them arguments just as you would with any function. But on the other hand they are part of the surrounding function or method. As I wrote the sample code above I didn’t think of the block as a separate function - I thought of the block’s code as just part of the simple, top level script that printed a string 10 times.

How does this work internally? Does Ruby internally implement blocks as separate functions? Or as part of the surrounding function? Let’s step through the example above, slowly, and see what happens inside of Ruby when you call a block.

In this example when Ruby executes the first line of code, as I explained in Chapter 2, YARV will store the local variable <span class="code">str</span> on it’s internal stack, and save it’s location in the DFP pointer located in the current <span class="code">rb_control_frame_t</span> structure\*. (\*footnote: If the outer code was located inside a function or method then the DFP would point to the stack frame as shown, but if the outer code was located in the top level scope of your Ruby program, then Ruby would use dynamic access to save the variable in the TOPLEVEL_BINDING environment instead - more on this in section 4.3. Regardless the DFP will always indicate the location of the <span class="code">str</span> variable.)

<img src="http://patshaughnessy.net/assets/2012/9/18/save-local-variable.png"/>

Next Ruby will come to the <span class="code">10.times do</span> call. Before executing the actual iteration - before calling the <span class="code">times</span> method - Ruby will first create and initialize a new <span class="code">rb_block_t</span> structure to represent the block. Ruby needs to create the block structure now, since the block is really just another argument to the <span class="code">times</span> method:

<img src="http://patshaughnessy.net/assets/2012/9/18/new-block.png"/>

To do this, Ruby copies the current value of the DFP, the dynamic frame pointer, into the new block. In other words, Ruby saves away the location of the current stack frame in the new block.

Next Ruby will proceed to call the <span class="code">times</span> method on the object <span class="code">10</span>, an instance of the <span class="code">Fixnum</span> class. While doing this, YARV will create a new frame on its internal stack. Now we have two stack frames: on the top is the new stack frame for the <span class="code">Fixnum.times</span> method, and below is the original stack frame used by the top level function:

<img src="http://patshaughnessy.net/assets/2012/9/18/fixnum-stack.png"/>

Ruby implements the <span class="code">times</span> method internally using its own C code - it’s a built in method - but Ruby implements it the same way you probably would in Ruby. Ruby starts to iterate over the numbers 0, 1, 2, etc., up to 9, and calls <span class="code">yield</span> once for each of these integers. Finally, the code that implements <span class="code">yield</span> internally actually calls the block each time through the loop, pushing a third\* frame onto the top of the stack for the code inside the block to use: (*footnote: Ruby actually pushes an extra, internal stack frame whenever you call yield before actually calling the block, so strictly speaking there should be four stack frames in this diagram. I only show three for the sake of clarity.)

<img src="http://patshaughnessy.net/assets/2012/9/18/executing-block.png"/>

Here on the left we now have three stack frames:

<ul>
<li>On the top is the new stack frame for the block, containing the <span class="code">str2</span> variable.</li>
<li>In the middle is the stack frame used by the internal C code that implements the <span class="code">Fixnum.times</span> method.</li>
<li>And at the bottom is the original function’s stack frame, containing the <span class="code">str</span> variable from the outer scope.</li>
</ul>

While creating the new stack frame at the top, Ruby’s internal <span class="code">yield</span> code copies the DFP from the block into the new stack frame. Now the code inside the block can access both it’s own local variables, via the <span class="code">rb_control_frame</span> structure as usual, and indirectly the variables from the parent scope, via the DFP pointer using dynamic variable access, as I explained in Chapter 2. Specifically this allows the <span class="code">puts</span> statement to access the <span class="code">str</span> variable from the parent scope.

To summarize, we have seen now that Ruby’s <span class="code">rb_block_t</span> structure contains two important values:
a pointer to a snippet of YARV code instructions, and
a pointer to a location on YARV’s internal stack, the location that was at the top of the stack when the block was created:

<img src="http://patshaughnessy.net/assets/2012/9/18/code-and-stack.png"/>

At first glance this seems like a very technical, unimportant detail. This is obviously a behavior we expect Ruby blocks to exhibit, and the DFP seems to be just another minor, uninteresting part of Ruby’s internal implementation of blocks.

Or is it? I believe the DFP is actually a profound, important part of Ruby internals. The DFP is the basis for Ruby’s implementation of “closures.” Here’s how Sussman and Steele defined the term “closure” in their 1975 paper <i><a href="http://en.wikisource.org/wiki/Scheme:_An_Interpreter_for_Extended_Lambda_Calculus">Scheme: An Interpreter for Extended Lambda Calculus</a></i>:

<br/>
<blockquote>
In order to solve this problem we introduce the notion of a closure [11, 14] which is a data structure containing a lambda expression, and an environment to be used when that lambda expression is applied to arguments.
</blockquote>

Reading this again carefully, a closure is defined to be the combination of:

<ul>
<li>A “lambda expression,” i.e. a function that takes a set of arguments, and</li>
<li>An environment to be used when calling that lambda or function.</li>
</ul>

I’ll have more context and information about “lambda expressions” and how Ruby’s borrowed the <span class="code">lambda</span> keyword from Lisp in section 4-2, but for now take another look at the internal <span class="code">rb_block_t</span> structure:

<img src="http://patshaughnessy.net/assets/2012/9/18/code-and-stack.png"/>

Notice that this structure meets the definition of a closure Sussman and Steele wrote back in 1975:

<ul>
<li><span class="code">iseq</span> is a pointer to a lambda expression, i.e. a function or code snippet, and</li>
<li><span class="code">DFP</span> is a pointer to the environment to be used when calling that lambda or function, i.e. a pointer to the surrounding stack frame.</li>
</ul>

Following this train of thought, we can see that blocks are Ruby’s implementation of closures. Ironically blocks, one of the features that in my opinion makes Ruby so elegant and natural to read - so modern and innovative - is based on research and work done at least 20 years before Ruby was ever invented!

<div class="yellow">

<p><b>
[ Note: In Ruby Under a Microscope I won't show or discuss any C code directly, except for optional sections that are called out with a different background color like this. ]
</b></p>

<br/>

In Ruby 1.9 and later you can find the actual definition of the <span class="code">rb_block_t</span> structure in the vm_core.h file. Here it is:

<br/>
<br/>
<pre type="c">
typedef struct rb_block_struct {
    VALUE self;			/* share with method frame if it's only block */
    VALUE *lfp;			/* share with method frame if it's only block */
    VALUE *dfp;			/* share with method frame if it's only block */
    rb_iseq_t *iseq;
    VALUE proc;
} rb_block_t;
</pre>

<br/>
You can see the <span class="code">iseq</span> and <span class="code">DFP</span> values I described above, along with a few other values:
<ul>
<li><span class="code">self</span>: As we’ll see in the next sections when I cover lambdas, procs and bindings, the value the <span class="code">self</span> pointer had when the block was first referred to is also an important part of the closure’s environment. Ruby executes block code inside the same object context the code outside the block had.</li>
<li><span class="code">lfp</span>: It turns out blocks also contain a local frame pointer, along with the dynamic frame pointer. However, Ruby doesn’t use local variable access inside of blocks; it doesn’t use the set/getlocal YARV instructions inside of blocks. Instead, Ruby uses this LFP for internal, technical reasons and not to access local variables.</li>
<li><span class="code">proc</span>: Finally, Ruby uses this value when it creates a proc object from a block. As we’ll see in the next section, procs and blocks are closely related.</li>
</ul>

Right above the definition of <span class="code">rb_block_t</span> in vm_core.h you’ll see the <span class="code">rb_control_frame_t</span> structure defined:

<br/>
<br/>
<pre type="c">
typedef struct {
    VALUE *pc;			/* cfp[0] */
    VALUE *sp;			/* cfp[1] */
    VALUE *bp;			/* cfp[2] */
    rb_iseq_t *iseq;	/* cfp[3] */
    VALUE flag;			/* cfp[4] */
    VALUE self;			/* cfp[5] / block[0] */
    VALUE *lfp;			/* cfp[6] / block[1] */
    VALUE *dfp;			/* cfp[7] / block[2] */
    rb_iseq_t *block_iseq;	/* cfp[8] / block[3] */
    VALUE proc;			/* cfp[9] / block[4] */
    const rb_method_entry_t *me;/* cfp[10] */
} rb_control_frame_t;
</pre>
<br/>

Notice that this C structure also contains all of the same values the <span class="code">rb_block_t</span> structure did: everything from <span class="code">self</span> down to <span class="code">proc</span>. The fact these two structures share the same values is actually one of the interesting, but confusing, optimizations Ruby uses internally to speed things up a bit. Whenever you refer to a block for the first time by passing it into a method call, as I explained above Ruby creates a new <span class="code">rb_block_t</span> structure and copies values such as the LFP from the current <span class="code">rb_control_frame_t</span> structure into it. However, by making the members of these two structures similar &ndash; <span class="code">rb_block_t</span> is a subset of <span class="code">rb_control_frame_t</span> &ndash; Ruby is able to avoid creating a new <span class="code">rb_block_t</span> structure and instead sets the pointer to the new block to refer to the common portion of the <span class="code">rb_control_frame_t</span> structure. In other words, instead of allocating new memory to hold a new <span class="code">rb_block_t</span> structure, Ruby simple passes around a pointer to the middle of the <span class="code">rb_control_frame_t</span> structure. This is very confusing, but does avoid unnecessary calls to malloc, and speeds up the process of creating blocks.
</div>

## Experiment 4-1: Which is faster: a while-loop or passing a block to each?

… read it in the [finished eBook](http://patshaughnessy.net/ruby-under-a-microscope).

## Lambdas and Procs: treating code as a first class citizen

… read it in the [finished eBook](http://patshaughnessy.net/ruby-under-a-microscope).

## Experiment 4-2: Lambda/Proc performance - how long does it take Ruby to copy a stack frame to the heap?

… read it in the [finished eBook](http://patshaughnessy.net/ruby-under-a-microscope).

## Bindings: A closure environment without a function

… read it in the [finished eBook](http://patshaughnessy.net/ruby-under-a-microscope).

## Experiment 4-3: Exploring the TOPLEVEL_BINDING object.

… read it in the [finished eBook](http://patshaughnessy.net/ruby-under-a-microscope).

## Closures in JRuby

… read it in the [finished eBook](http://patshaughnessy.net/ruby-under-a-microscope).

## Closures in Rubinius

… read it in the [finished eBook](http://patshaughnessy.net/ruby-under-a-microscope).

