title: "How to integrate Amazon Simple Pay into a Rails 3 app"
date: 2010/11/20

<p>Amazon Payments is a great alternative to PayPal or Active Merchant - users of your site can pay you via their Amazon account and like with PayPal you don&rsquo;t need to store credit card information in your own database. Another plus is they provide a special service tailored to non-profits, allowing users to make donations. We used this for ReliefHub, an online charity site <a href="http://patshaughnessy.net/2010/11/1/reliefhub-hackfest-helping-orphanages-in-haiti">I volunteered to work on last month</a>.</p>
<p>Today I&rsquo;ll show you how to create a simple scaffolding Rails site with an Amazon Payments button that looks like this:</p>
<p><img src="http://patshaughnessy.net/assets/2010/11/20/listing_payments.png"></p>
<p>It will actually accept payments: users will be able to enter an amount, click &ldquo;Pay Now&rdquo; and then confirm the transaction on the Amazon site. Later they will be redirected back to the Rails site, and their transaction will appear in the index scaffolding view.</p>
<p>But first the hard part: dealing with the poor Amazon documentation, confusing admin web pages and numerous sign up forms. Bear with me! If you can suffer through the pain and confusion of the Amazon sign up process, it turns out to be quite easy to use Amazon Payments in a Rails 3 site.</p>
<h2>Step 1: Sign up for Amazon Web Services</h2>
<p>If you already use S3, EC2 or other Amazon web services, then skip to step 2. Otherwise, go to: <a href="http://aws.amazon.com">http://aws.amazon.com</a>, click &ldquo;Sign Up Now&rdquo; and enter your Amazon email address and password. Then you need to create a new web services account by filling out the &ldquo;Contact Information&rdquo; form and agreeing to the &ldquo;AWS Customer Agreement.&rdquo;</p>
<h2>Step 2: Sign up for Amazon Payments</h2>
<p>There are actually two different Amazon Payment Services sites: sandbox and production. The sandbox allows you to test transactions from your web site without using real money. You will have separate account IDs for Amazon Payments sandbox and production.</p>
<p>For now, let&rsquo;s start with Amazon Payments Sandbox; open: <a href="https://payments-sandbox.amazon.com">https://payments-sandbox.amazon.com</a></p>
<p>You should see the word &ldquo;sandbox&rdquo; in the Amazon Payments logo at the top left:</p>
<p><img src="http://patshaughnessy.net/assets/2010/11/20/payment_sandbox.png"></p>
<p>Later when you are ready to deploy your Rails app and want to collect real money you&rsquo;ll need to repeat steps 2, 3 and 4 using the production Amazon site: <a href="https://payments.amazon.com">https://payments.amazon.com</a></p>
<p>From the Amazon Payments home page, click &ldquo;Your Account&rdquo; and then login again using your Amazon email address and password. After you enter your Amazon credentials, you&rsquo;ll need to sign up and enter your contact information again, this time for Amazon Payments. Once you sign up, you should be redirected to your account page, which will display your account balance and recent transaction information.</p>
<h2>Step 3: Sign up for the Amazon Flexible Payment Service (FPS)</h2>
<p>Now for the confusing part: <i>you need to sign up for a third time</i>; this time for the &ldquo;Amazon Flexible Payments Service.&rdquo; To do this, click &ldquo;Developers&rdquo; and then click the &ldquo;Sign Up For Amazon FPS&rdquo; button on the right side of the page:</p>
<p><img src="http://patshaughnessy.net/assets/2010/11/20/amazon_fps_signup.png"></p>
<p>Now that you&rsquo;ve signed up for Amazon FPS <b>you should promptly decide not to use it</b>. It&rsquo;s very confusing, complex and provides little additional value compared to Amazon Simple Pay. You might want to consider Amazon FPS if you need to make payments programmatically on a scheduled basis with no user interface or buttons. If that&rsquo;s the case, check out <a href="http://github.com/tylerhunt/remit">the Remit gem on github</a>, which will make Amazon FPS/Rails integration easier.</p>
<p>Amazon Simple Pay, on the other hand, is a simple, straightforward way to add payment forms to your Rails site. That is, simple except for the sign up process and confusing documentation!</p>
<h2>Step 4: Disable the &ldquo;Sign the buttons&rdquo; option</h2>
<p>To improve security, Amazon Simple Pay normally requires that payment requests include an encrypted hash of the form values called a &ldquo;signature.&rdquo; For today, you should disable this to keep things simple. Later in a future post I&rsquo;ll show how to generate the signature value in your Rails site, allowing you to enable this feature.</p>
<ul>
<li>Return to the <a href="https://payments-sandbox.amazon.com">https://payments-sandbox.amazon.com</a> home page</li>
<li>Go to: Your Account -&gt; Edit My Account Settings -&gt; Manage Developer and Seller Preferences</li>
<li>Uncheck &ldquo;Sign the buttons&rdquo; if necessary<br><img src="http://patshaughnessy.net/assets/2010/11/20/sign_the_buttons.png"></li>
<li>Click &ldquo;Confirm&rdquo; at the bottom of the form if necessary.</li>
</ul>
<h2>Step 5: Create an Amazon Simple Pay button</h2>
<p>Amazon Simple Pay is very easy to use since you can actually generate the HTML you need to create a payment form/button right on the Amazon Payments site. But it took me hours of clicking and searching to find the Amazon Payments admin page that allows you to generate the button.</p>
<p>Here&rsquo;s a link to the create button form: <a href="https://payments-sandbox.amazon.com/sdui/sdui/standardbutton">https://payments-sandbox.amazon.com/sdui/sdui/standardbutton</a></p>
<p>Bookmark this or you&rsquo;ll never find it again! There are other pages you can use to generate &ldquo;Donate&rdquo; buttons for non-profits, or other types of buttons. Now by filling out this form, you can create a payment form using one of the Amazon payment buttons, for example:</p>
<p><img src="http://patshaughnessy.net/assets/2010/11/20/pay_now.png"></p>
<p>Here&rsquo;s what to do:</p>
<ul>
  <li>Click the link above</li>
  <li>Sign in again with your Amazon credentials if necessary</li>
  <li>Enter any values you want for Amount, Description, and Reference since we&rsquo;re going to change them later in our Rails code.</li>
  <li>Enter any valid URL you want for &ldquo;Return URL&rdquo;. Later we&rsquo;ll replace this with a Rails route.</li>
  <li>Leave User Abandon URL and URL for Instant Payment Notification blank.</li>
  <li>Leave the other default values unchanged.</li>
  <li>Select one of the Amazon buttons; later you can replace their button image with your own button, of course.</li>
  <li>Next, click &ldquo;Generate HTML&rdquo; and fix any form validations errors you get.</li>
</ul>
<p>Now you should see HTML similar to this in the text area:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="ta">&lt;form</span> <span class="an">action</span>=<span class="s"><span class="dl">&quot;</span><span class="k">https://authorize.payments-sandbox.amazon.com/pba/paypipeline</span><span class="dl">&quot;</span></span> <span class="an">method</span>=<span class="s"><span class="dl">&quot;</span><span class="k">post</span><span class="dl">&quot;</span></span><span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">immediateReturn</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">1</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">collectShippingAddress</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">signatureVersion</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">2</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">signatureMethod</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">HmacSHA256</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">accessKey</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">11SEM03K88SD016FS1G2</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">referenceId</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">test</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">amount</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">USD 10</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">signature</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">avlT64an02VS0drWiNZj36rGz88JwvFmETKveYfzjJk=</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">isDonationWidget</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">description</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">test</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">amazonPaymentsAccountId</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">KWKCCYNIVXJL42AE3FVCZCA65I3522JBV4EJVX</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">returnUrl</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">http://test.com/return_url</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">processImmediate</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">1</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">cobrandingStyle</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">logo</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">image</span><span class="dl">&quot;</span></span> <span class="an">src</span>=<span class="s"><span class="dl">&quot;</span><span class="k">http://g-ecx.images-amazon.com/images/G/01/asp/beige_small_paynow_withmsg_whitebg.gif</span><span class="dl">&quot;</span></span> <span class="an">border</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span><span class="ta">&gt;</span>
<span class="ta">&lt;/form&gt;</span></pre></div>
</div><br/><br/>
<p>Copy and paste the HTML into a text file somewhere and hold onto it. By the way, there are two important values hidden in this HTML: your AWS access key, and also your Amazon Payments account ID. You&rsquo;ll want to make a note of these values and save them somewhere also.</p>
<h2>Step 6: Write a Rails app to collect payments</h2>
<p>Thank goodness; we&rsquo;re done with the Amazon Payments user interface! Now the fun part: let&rsquo;s build a Rails 3.0 sample app to collect payments using the Amazon button we just generated:</p>
<div class="CodeRay">
  <div class="code"><pre>$ rails new sample_app
    <span class="s">create</span>  
    <span class="s">create</span>  README
    <span class="s">create</span>  Rakefile
    <span class="s">create</span>  config.ru
    <span class="s">create</span>  .gitignore
    <span class="s">create</span>  Gemfile
    
$ cd sample_app</pre></div>
</div><br/>
<p>And let&rsquo;s create a model to hold our payment transactions - I&rsquo;ll store just the amount and the transaction id for today. You could also store the user&rsquo;s email, shipping address, etc.
<div class="CodeRay">
  <div class="code"><pre>$ rails generate scaffold payment amount:float transaction_id:string
      invoke  active_record
      <span class="s">create</span>    db/migrate/20101119212749_create_payments.rb
      <span class="s">create</span>    app/models/payment.rb
      invoke    test_unit
      <span class="s">create</span>      test/unit/payment_test.rb
      <span class="s">create</span>      test/fixtures/payments.yml
       <span class="s">route</span>  resources :payments
      invoke  scaffold_controller
etc...

$ rake db:migrate</pre></div>
</div><br/>
<p>Now edit app/views/payments/index.html.erb and paste in the HTML from Amazon Simple Pay at the top:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="ta">&lt;form</span> <span class="an">action</span>=<span class="s"><span class="dl">&quot;</span><span class="k">https://authorize.payments-sandbox.amazon.com/pba/paypipeline</span><span class="dl">&quot;</span></span> <span class="an">method</span>=<span class="s"><span class="dl">&quot;</span><span class="k">post</span><span class="dl">&quot;</span></span><span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">immediateReturn</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">1</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">collectShippingAddress</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">signatureVersion</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">2</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">signatureMethod</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">HmacSHA256</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">accessKey</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">11SEM03K88SD016FS1G2</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">referenceId</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">ref</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">amount</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">USD 10</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">signature</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">avlT64an02VS0drWiNZj36rGz88JwvFmETKveYfzjJk</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">isDonationWidget</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">description</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">desc</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">amazonPaymentsAccountId</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">KWKCCYNIVXJL42AE3FVCZCA65I3522JBV4EJVX</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">returnUrl</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">http://test.com/return_url</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">processImmediate</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">1</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">cobrandingStyle</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">logo</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">image</span><span class="dl">&quot;</span></span> <span class="an">src</span>=<span class="s"><span class="dl">&quot;</span><span class="k">http://g-ecx.images-amazon.com/images/G/01/asp/beige_small_paynow_withmsg_whitebg.gif</span><span class="dl">&quot;</span></span> <span class="an">border</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span><span class="ta">&gt;</span>
<span class="ta">&lt;/form&gt;</span>

<span class="ta">&lt;h1&gt;</span>Listing payments<span class="ta">&lt;/h1&gt;</span>

<span class="ta">&lt;table&gt;</span>
  <span class="ta">&lt;tr&gt;</span>
    <span class="ta">&lt;th&gt;</span>Amount<span class="ta">&lt;/th&gt;</span>
    <span class="ta">&lt;th&gt;</span>Transaction<span class="ta">&lt;/th&gt;</span>

etc...
</pre></div>
</div><br/>
<p>Aside from the important accessKey and the amazonPaymentsAccountId values, you might have noticed there&rsquo;s another value called &ldquo;signature,&rdquo; along with &ldquo;signatureVersion&rdquo; and &ldquo;signatureMethod.&rdquo; As I mentioned above, the signature is actually an encrypted hash of all of the other form values. When users submit the form, Amazon will check that all of the form parameters are valid by comparing them against the signature value. The problem is that for Amazon &ldquo;valid&rdquo; means equal to what you entered on the Amazon create button form earlier - not what is actually valid for your site or business rules.</p>
<p>To make the Amazon Simple Pay form work with different payment amounts and with different return URLs (e.g. &ldquo;localhost:3000&rdquo; vs. your production hostname) we can&rsquo;t use this signature security scheme... at least not for now. Later in a future blog post I&rsquo;ll show a way to dynamically generate the signature value each time the payment form is submitted.</p>
<p>For now, let&rsquo;s just delete the signature related values; remember we unchecked &ldquo;Sign the buttons&rdquo; earlier in the Amazon Payments site; setting this to false allows us to remove the signature value.</p>
<p>Also, you should change the Return URL to: &ldquo;http://localhost:3000/confirm_payment.&rdquo; This tells Amazon payments how to redirect the user back to our site after they have confirmed the payment.</p>
<p>Here&rsquo;s what my payment index view looks like now:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="ta">&lt;form</span> <span class="an">action</span>=<span class="s"><span class="dl">&quot;</span><span class="k">https://authorize.payments-sandbox.amazon.com/pba/paypipeline</span><span class="dl">&quot;</span></span> <span class="an">method</span>=<span class="s"><span class="dl">&quot;</span><span class="k">post</span><span class="dl">&quot;</span></span><span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">immediateReturn</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">1</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">collectShippingAddress</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">accessKey</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">11SEM03K88SD016FS1G2</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">referenceId</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">ref</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">amount</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">USD 10</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">isDonationWidget</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">description</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">desc</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">amazonPaymentsAccountId</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">KWKCCYNIVXJL42AE3FVCZCA65I3522JBV4EJVX</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class='container'><span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">returnUrl</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">http://localhost:3000/confirm_payment</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span><span class='overlay'></span></span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">processImmediate</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">1</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">hidden</span><span class="dl">&quot;</span></span> <span class="an">name</span>=<span class="s"><span class="dl">&quot;</span><span class="k">cobrandingStyle</span><span class="dl">&quot;</span></span> <span class="an">value</span>=<span class="s"><span class="dl">&quot;</span><span class="k">logo</span><span class="dl">&quot;</span></span> <span class="ta">&gt;</span>
  <span class="ta">&lt;input</span> <span class="an">type</span>=<span class="s"><span class="dl">&quot;</span><span class="k">image</span><span class="dl">&quot;</span></span> <span class="an">src</span>=<span class="s"><span class="dl">&quot;</span><span class="k">http://g-ecx.images-amazon.com/images/G/01/asp/beige_small_paynow_withmsg_whitebg.gif</span><span class="dl">&quot;</span></span> <span class="an">border</span>=<span class="s"><span class="dl">&quot;</span><span class="k">0</span><span class="dl">&quot;</span></span><span class="ta">&gt;</span>

<span class="ta">&lt;/form&gt;</span>

<span class="ta">&lt;h1&gt;</span>Listing payments<span class="ta">&lt;/h1&gt;</span>

<span class="ta">&lt;table&gt;</span>
  <span class="ta">&lt;tr&gt;</span>
    <span class="ta">&lt;th&gt;</span>Amount<span class="ta">&lt;/th&gt;</span>
    <span class="ta">&lt;th&gt;</span>Transaction<span class="ta">&lt;/th&gt;</span>

etc...
</pre></div>
</div>
<br/>
<p>Next let&rsquo;s add a new route for confirming payments; edit config/routes.rb:</p>
  <div class="CodeRay">
    <div class="code"><pre><span class="co">SampleApp</span>::<span class="co">Application</span>.routes.draw <span class="r">do</span>

  resources <span class="sy">:payments</span>
  <span class="container">match <span class="s"><span class="dl">'</span><span class="k">confirm_payment</span><span class="dl">'</span></span> =&gt; <span class="s"><span class="dl">'</span><span class="k">payments#confirm</span><span class="dl">'</span></span><span class='overlay'></span></span>

etc...</pre></div>
  </div><br>
<p>And now let&rsquo;s write a new action in the payments controller for this; edit app/controllers/payments_controller.rb:</p>
  <div class="CodeRay">
    <div class="code"><pre><span class="r">class</span> <span class="cl">PaymentsController</span> &lt; <span class="co">ApplicationController</span>

  <span class="r">def</span> <span class="fu">confirm</span>
    <span class="iv">@payment</span> = <span class="co">Payment</span>.new(
      <span class="sy">:transaction_amount</span> =&gt; params[<span class="sy">:transactionAmount</span>],
      <span class="sy">:transaction_id</span>     =&gt; params[<span class="sy">:transactionId</span>]
    )
    <span class="r">if</span> <span class="iv">@payment</span>.save
      redirect_to(<span class="iv">@payment</span>, <span class="sy">:notice</span> =&gt; <span class="s"><span class="dl">'</span><span class="k">Payment was successfully created.</span><span class="dl">'</span></span>)
    <span class="r">else</span>
      redirect_to <span class="sy">:action</span> =&gt; <span class="s"><span class="dl">&quot;</span><span class="k">index</span><span class="dl">&quot;</span></span>
    <span class="r">end</span>
  <span class="r">end</span>
  
etc...</pre></div>
  </div><br/>
<p>This looks similar to the create action we got from the scaffolding generator. This action will be called when users return from the Amazon Payments site, and will record the transaction in our application&rsquo;s payments table. More on this in a moment.</p>
<p>Finally in the payment model we need a bit of code to parse the transaction amount string passed back from Amazon:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">class</span> <span class="cl">Payment</span> &lt; <span class="co">ActiveRecord</span>::<span class="co">Base</span>

  validates_presence_of <span class="sy">:amount</span>
  validates_presence_of <span class="sy">:transaction_id</span>

  <span class="r">def</span> <span class="fu">transaction_amount=</span>(currency_and_amount)
    currency = parse(currency_and_amount).first
    <span class="r">if</span> currency == <span class="s"><span class="dl">'</span><span class="k">USD</span><span class="dl">'</span></span>
      amount = parse(currency_and_amount).last.to_f
    <span class="r">else</span>
      amount = currency.to_f
    <span class="r">end</span>
    <span class="pc">self</span>.amount = amount <span class="r">unless</span> amount == <span class="fl">0.0</span>
  <span class="r">end</span>

  <span class="r">def</span> <span class="fu">parse</span>(currency_and_amount)
    <span class="iv">@parsed</span> ||= currency_and_amount.split
  <span class="r">end</span>
<span class="r">end</span>
</pre></div>
</div><br/>
<p>The &ldquo;transaction amount&rdquo; parameter from Amazon may contain a prefix &ldquo;USD&rdquo; specifying the currency. However, sometimes this is not present. The code above parses the string as necessary in a virtual attribute &ldquo;transaction_amount&rdquo; and then saves the corresponding amount as the &ldquo;amount&rdquo; float value in the database. The Payment model also validates that both amount and transaction id are specified by Amazon in order to consider the payment valid.</p>
<p>Now let&rsquo;s fire up the app and try it out!</p>
<p><img src="http://patshaughnessy.net/assets/2010/11/20/listing_payments_empty.png"></p>
<p>If you click the &ldquo;Pay Now&rdquo; button you&rsquo;ll be redirected to Amazon:</p>
<p><img src="http://patshaughnessy.net/assets/2010/11/20/amazon_sign_in.png"></p>
<p>Your personal or business name should appear at the top left instead of my name. Next sign to Amazon with a different account and confirm the payment... you can&rsquo;t use the same account you used to setup Amazon Simple Pay, since Amazon doesn&rsquo;t allow someone to pay themselves. Also remember &ldquo;sandbox&rdquo; appeared in the form action URL, so there&rsquo;s no need to worry about spending real money yet!</p>
<p>After you click &ldquo;Confirm&rdquo; you&rsquo;ll be redirected back to our Rails sample app, to the page we gave Amazon in the &ldquo;returnUrl&rdquo; form field. Now you should see:</p>
<p><img src="http://patshaughnessy.net/assets/2010/11/20/payment_successful.png"></p>
<p>The transaction id and amount are now saved in the transactions table, which you can use for reporting or for other purposes in your user interface. This all worked as follows:
<ul>
  <li>Amazon sent a GET request to the URL we provided in the &ldquo;returnUrl&rdquo; field in the payment form.</li>
  <li>In this request, Amazon passed information about the transaction, such as the amount, the transaction id and other information like the buyer&rsquo;s email address, name, shipping address - and a signature hash is also provided in case you want to be sure this request is from Amazon.</li>
  <li>Since our returnUrl was set to confirm_payment_url, our confirm action was called in the PaymentsController above.</li>
  <li>This creates a new Payment record, setting just the amount and transaction id values in this example.</li>
  <li>Finally, the Payment model parses the &ldquo;transactionAmount&rdquo; value from Amazon as I explained above.</li>
</ul></p>
<h2>Step 7: Using a helper to generate the payment form easily</h2>
<p>Of course our view code is very ugly - all of that complex HTML is hard to read and confusing, and also if we want more than one button in our app we will have to repeat all of the HTML. The best thing to do render the payment form using a helper. This code will do the trick - save this code in app/helpers/amazon_simple_pay_helper.rb:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="r">module</span> <span class="cl">AmazonSimplePayHelper</span>

  <span class="co">AMAZON_ACCESS_KEY</span> = <span class="s"><span class="dl">'</span><span class="k">11SEM03K88SD016FS1G2</span><span class="dl">'</span></span>

  <span class="co">AMAZON_PAYMENTS_ACCOUNT_ID</span> = <span class="s"><span class="dl">'</span><span class="k">KWKCCYNIVXJL42AE3FVCZCA65I3522JBV4EJVX</span><span class="dl">'</span></span>

  <span class="r">def</span> <span class="fu">amazon_simple_pay_form_tag</span>(options = {}, &amp;block)
    sandbox = <span class="s"><span class="dl">'</span><span class="k">-sandbox</span><span class="dl">'</span></span> <span class="r">unless</span> <span class="co">Rails</span>.env == <span class="s"><span class="dl">'</span><span class="k">production</span><span class="dl">'</span></span>
    pipeline_url = <span class="s"><span class="dl">&quot;</span><span class="k">https://authorize.payments</span></span><span class="il"><span class="idl">#{</span>sandbox<span class="idl">}</span></span><span class="s">.amazon.com/pba/paypipeline</span><span class="dl">&quot;</span></span> 
    html_options = { <span class="sy">:action</span> =&gt; pipeline_url, <span class="sy">:method</span> =&gt; <span class="sy">:post</span> }.merge(options)
    content = capture(&amp;block)
    output = <span class="co">ActiveSupport</span>::<span class="co">SafeBuffer</span>.new
    output.safe_concat(tag(<span class="sy">:form</span>, html_options, <span class="pc">true</span>))
    output &lt;&lt; content
    output.safe_concat(hidden_field_tag(<span class="s"><span class="dl">'</span><span class="k">accessKey</span><span class="dl">'</span></span>, <span class="co">AMAZON_ACCESS_KEY</span>))
    output.safe_concat(hidden_field_tag(<span class="s"><span class="dl">'</span><span class="k">amazonPaymentsAccountId</span><span class="dl">'</span></span>, <span class="co">AMAZON_PAYMENTS_ACCOUNT_ID</span>))
    output.safe_concat(hidden_field_tag(<span class="s"><span class="dl">'</span><span class="k">immediateReturn</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">1</span><span class="dl">'</span></span>))
    output.safe_concat(hidden_field_tag(<span class="s"><span class="dl">'</span><span class="k">processImmediate</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">1</span><span class="dl">'</span></span>))
    output.safe_concat(hidden_field_tag(<span class="s"><span class="dl">'</span><span class="k">cobrandingStyle</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">logo</span><span class="dl">'</span></span>))
    output.safe_concat(hidden_field_tag(<span class="s"><span class="dl">'</span><span class="k">returnUrl</span><span class="dl">'</span></span>, confirm_payment_url))
    output.safe_concat(<span class="s"><span class="dl">&quot;</span><span class="k">&lt;/form&gt;</span><span class="dl">&quot;</span></span>)
  <span class="r">end</span>

<span class="r">end</span></pre></div>
</div><br/>
<p>This helper uses the same code the standard &ldquo;form_tag&rdquo; Rails view helper does, except that it provides the additional fields and attributes required by Amazon. Here&rsquo;s how it works:</p>
<ul>
  <li>First it calculates the form action URL - the amazon payments sandbox URL from the HTML above. &ldquo;Sandbox&rdquo; is included in the URL unless your are running on your production server.</li>
  <li>It creates a SafeBuffer - this prevents Rails 3 from escaping the html.</li>
  <li>Then it writes the new form tag into this buffer with the Amazon form action URL and post method attributes.</li>
  <li>Next it yields to the provided block and appends whatever HTML is returned, allowing you to use a block in your view as usual.</li>
  <li>Finally it adds the other form fields that Amazon requires. &ldquo;returnUrl&rdquo; is still set to the confirm payment route we created earlier.</li>
  <li>You should move the two constants AMAZON_ACCESS_KEY and AMAZON_PAYMENTS_ACCOUNT_ID into one of your environment config files or even better into a YAML file somewhere.</li>
</ul>
<p>Since this works like a normal form_tag, my view code is now much simpler and also familiar; it looks like any other Rails form:</p>
<div class="CodeRay">
  <div class="code"><pre><span class="il"><span class="idl">&lt;%=</span> amazon_simple_pay_form_tag <span class="r">do</span> <span class="idl">%&gt;</span></span>
  $ <span class="il"><span class="idl">&lt;%=</span> text_field_tag <span class="s"><span class="dl">'</span><span class="k">amount</span><span class="dl">'</span></span> <span class="idl">%&gt;</span></span>&lt;br/&gt;&lt;br/&gt;
  <span class="il"><span class="idl">&lt;%=</span> hidden_field_tag <span class="s"><span class="dl">'</span><span class="k">referenceId</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">something</span><span class="dl">'</span></span> <span class="idl">%&gt;</span></span>
  <span class="il"><span class="idl">&lt;%=</span> hidden_field_tag <span class="s"><span class="dl">'</span><span class="k">description</span><span class="dl">'</span></span>, <span class="s"><span class="dl">'</span><span class="k">something else</span><span class="dl">'</span></span> <span class="idl">%&gt;</span></span>
  <span class="il"><span class="idl">&lt;%=</span> image_submit_tag(<span class="s"><span class="dl">'</span><span class="k">/images/pay_me.gif</span><span class="dl">'</span></span>) <span class="idl">%&gt;</span></span>
<span class="il"><span class="idl">&lt;%</span> <span class="r">end</span> <span class="idl">%&gt;</span></span></pre></div>
</div><br/>
<p>Here I&rsquo;ve provided a different button image, and allowed the user to enter any payment amount with a real text field. I&rsquo;ve also specified the reference id and description values here, since they are likely related to business rules and/or objects used elsewhere on this view page. To finish this off, I probably need to add some JQuery validation on the amount field.</p>
<p>Next time I&rsquo;ll show how to sign the form dynamically before it&rsquo;s submitted to Amazon, giving you the security benefit of the &ldquo;Sign buttons&rdquo; Amazon feature.</p>
