---
title: "Rate Limiting in NestJS Using Throttler"
url: "https://feeds.telerik.com/link/23069/17200233/rate-limiting-nestjs-using-throttler"
date: "2025-10-30T22:12:17Z"
author: "Christian Nwamba"
feed_url: "https://feeds.telerik.com/blogs/productivity-debugging"
---
<p><span class="featured">Rate limiting helps force clients to consume resources responsibly. Here&rsquo;s how to use this technique to secure our web server and its resources from abuse.</span></p><p>Rate limiting is a technique that verifies clients consume the server&rsquo;s resources responsibly. It is essential for preventing malicious actors from abusing resources. This technique monitors each client&rsquo;s requests to API endpoints. If their requests exceed a particular limit&mdash;that is, breach a particular constraint within a time frame&mdash;the client is either blocked or their request is delayed, slowed down or silently ignored.</p><p><a target="_blank" href="https://nestjs.com/">NestJS</a> is a popular framework that makes it easy for developers to build enterprise-grade server-side applications and provides them with a suite of tools to solve common problems.</p><p>In this article, we will be exploring the package for rate limiting. We will first explore rate limiting and some of its core concepts and see how this tool can be used to solve common rate limiting problems in our NestJS applications. We will explore some interesting challenges when implementing rate limiting in real-world applications and examine configurations that this package provides to help us counter those problems.</p><h2 id="prerequisites">Prerequisites</h2><p>This guide assumes the reader is familiar with TypeScript, backend development using Node.js, and HTTP and RESTful APIs.</p><h2 id="project-setup">Project Setup</h2><p>Assuming you have the <a target="_blank" href="https://docs.nestjs.com/cli/overview">Nest CLI</a> installed, let&rsquo;s now set up a NestJS project by running the following command:</p><pre class=" language-shell"><code class="prism  language-shell">nest new rate-limiting
</code></pre><p>The application is created in a folder called <code class="inline-code">rate-limiting</code>. Feel free to pick a folder name of your choice.</p><p>Next, let&rsquo;s install @nestjs/throttler, which is the rate limiting package for NestJS.</p><pre class=" language-shell"><code class="prism  language-shell">cd rate-limiting
npm install @nestjs/throttler
</code></pre><h2 id="how-rate-limiting-works">How Rate Limiting Works</h2><p>Now, let&rsquo;s briefly describe how rate limiting works and highlight some of the core concepts powering it. Understanding this section will be important when we start using the <code class="inline-code">@nestjs/throttler</code> package.</p><ul><li>The resource owner defines the constraint on what is allowed&mdash;that is, the <strong>limit</strong>. Limits are time-bound. The time frame a limit is applied to is called a <strong>window</strong>, and limits are expressed as &ldquo;limit per window&rdquo; (e.g., five requests per second, one job per hour, etc.).</li><li>The resource owner then tracks the client&rsquo;s requests to access their resources. Tracking can be done based on some identifier (e.g., an ID of some database entity, the client&rsquo;s IP address, etc.). Tracking is a stateful operation, so the resource owner implements some <strong>storage</strong> mechanism to hold tracking information for clients. The storage containers to hold this information could be databases or in-memory storage.</li><li>If the client exceeds the defined limit, the resource owner responds either by blocking the request for some duration and returning a 429 response code or by throttling the request (i.e., ignoring or slowing down the response to the user&rsquo;s request).</li><li>The resource owner implements a rate limiting algorithm to make everything work. Common rate limiting algorithms include token bucket, leaky bucket, fixed window and sliding window algorithms.</li></ul><p>Examples of rate limiting in real-world applications include the following:</p><ul><li>A payment gateway may limit requests from clients trying to get the status of a transaction to, for example, one request per minute.</li><li>A cloud service like EAS limits users with free tier accounts to only create 30 builds for their mobile applications per month, with a concurrency limit of one.</li><li>A media manipulation service like Cloudinary constrains API consumers on a free account to upload a limited number of files in each request, with each file not exceeding a particular size.</li></ul><h2 id="the-nestjsthrottler-module">The @nestjs/throttler Module</h2><p>Let&rsquo;s now explore the contents of the rate limiting module we will be using:</p><ul><li>It provides an abstraction over all the complexity involved in rate limiting and provides us with a simple interface to set up only the rate limiting options we need, while it handles the rest.</li><li>It provides a wide range of options that allow us to configure limits, handle client tracking, storage, etc.</li><li>It is storage agnostic, so it allows us to optionally configure different storage mechanisms for tracking. By default, tracking is done using an in-memory store using the JS Map data structure. It also supports regular databases&mdash;the most popular option here is Redis, as we will see later.</li><li>If the client exceeds the defined limit, this module responds by blocking the client.</li><li>Although this guide will explore how it can be used in a RESTful API, it is also compatible with GraphQL and WebSockets.</li></ul><h2 id="setting-up-some-routes">Setting up Some Routes</h2><p>Normally, a web server will contain one or more endpoints. Let us define a few. Update your <code class="inline-code">app.controller.ts</code> file with the following:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> Controller<span class="token punctuation">,</span> Get <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
@<span class="token function">Controller</span><span class="token punctuation">(</span><span class="token string">"app"</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppController</span> <span class="token punctuation">{</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/a"</span><span class="token punctuation">)</span>
  <span class="token function">getA</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is A"</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/b"</span><span class="token punctuation">)</span>
  <span class="token function">getB</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is B"</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/b"</span><span class="token punctuation">)</span>
  <span class="token function">getC</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is C"</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>We defined three routes: <code class="inline-code">/app/a</code>, <code class="inline-code">/app/b</code> and <code class="inline-code">/app/c</code>. Verify that <code class="inline-code">AppController</code> is mounted in the <code class="inline-code">app.module.ts</code> file as shown below:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> Module <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> AppController <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"./app.controller"</span><span class="token punctuation">;</span>
@<span class="token function">Module</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
  imports<span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token punctuation">]</span><span class="token punctuation">,</span>
  controllers<span class="token punctuation">:</span> <span class="token punctuation">[</span>AppController<span class="token punctuation">]</span><span class="token punctuation">,</span>
  providers<span class="token punctuation">:</span> <span class="token punctuation">[</span><span class="token punctuation">]</span><span class="token punctuation">,</span>
<span class="token punctuation">}</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppModule</span> <span class="token punctuation">{</span><span class="token punctuation">}</span>
</code></pre><h2 id="add-rate-limiting-to-the-entire-app">Add Rate Limiting to the Entire App</h2><p>Update the <code class="inline-code">app.module.ts</code> file to look like this:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> Module <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> AppService <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"./app.service"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span>
  minutes<span class="token punctuation">,</span>
  seconds<span class="token punctuation">,</span>
  ThrottlerGuard<span class="token punctuation">,</span>
  ThrottlerModule<span class="token punctuation">,</span>
<span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/throttler"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> APP_GUARD <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/core"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> AppController <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"./app.controller"</span><span class="token punctuation">;</span>
@<span class="token function">Module</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
  imports<span class="token punctuation">:</span> <span class="token punctuation">[</span>
    ThrottlerModule<span class="token punctuation">.</span><span class="token function">forRoot</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
      throttlers<span class="token punctuation">:</span> <span class="token punctuation">[</span>
        <span class="token punctuation">{</span>
          name<span class="token punctuation">:</span> <span class="token string">"first"</span><span class="token punctuation">,</span>
          ttl<span class="token punctuation">:</span> <span class="token number">1000</span><span class="token punctuation">,</span>
          limit<span class="token punctuation">:</span> <span class="token number">1</span><span class="token punctuation">,</span>
          blockDuration<span class="token punctuation">:</span> <span class="token number">1000</span><span class="token punctuation">,</span>
        <span class="token punctuation">}</span><span class="token punctuation">,</span>
        <span class="token punctuation">{</span>
          name<span class="token punctuation">:</span> <span class="token string">"second"</span><span class="token punctuation">,</span>
          ttl<span class="token punctuation">:</span> <span class="token function">seconds</span><span class="token punctuation">(</span><span class="token number">10</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token comment">// 10000</span>
          limit<span class="token punctuation">:</span> <span class="token number">5</span><span class="token punctuation">,</span>
          blockDuration<span class="token punctuation">:</span> <span class="token function">seconds</span><span class="token punctuation">(</span><span class="token number">5</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
        <span class="token punctuation">}</span><span class="token punctuation">,</span>
        <span class="token punctuation">{</span>
          name<span class="token punctuation">:</span> <span class="token string">"third"</span><span class="token punctuation">,</span>
          ttl<span class="token punctuation">:</span> <span class="token function">minutes</span><span class="token punctuation">(</span><span class="token number">1</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token comment">// 60000</span>
          limit<span class="token punctuation">:</span> <span class="token number">25</span><span class="token punctuation">,</span>
        <span class="token punctuation">}</span><span class="token punctuation">,</span>
      <span class="token punctuation">]</span><span class="token punctuation">,</span>
      errorMessage<span class="token punctuation">:</span> <span class="token string">"too many requests!"</span><span class="token punctuation">,</span>
    <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  controllers<span class="token punctuation">:</span> <span class="token punctuation">[</span>AppController<span class="token punctuation">]</span><span class="token punctuation">,</span>
  providers<span class="token punctuation">:</span> <span class="token punctuation">[</span>
    AppService<span class="token punctuation">,</span>
    <span class="token punctuation">{</span>
      provide<span class="token punctuation">:</span> APP_GUARD<span class="token punctuation">,</span>
      useClass<span class="token punctuation">:</span> ThrottlerGuard<span class="token punctuation">,</span>
    <span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
<span class="token punctuation">}</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppModule</span> <span class="token punctuation">{</span><span class="token punctuation">}</span>
</code></pre><p>The <code class="inline-code">ThrottleModule</code> enables us to define the rate limiting options in our app. Using either its <code class="inline-code">forRoot()</code> or <code class="inline-code">forRootAsync()</code> overloaded methods, we pass an object defining the options we want. The most important property is one called <code class="inline-code">throttlers</code>&mdash;it holds an array of objects, each representing the rate limiting options we want. We can declare as many as required. In our case, we specified three of them.</p><p>Each throttle option has properties, but the only mandatory ones are <code class="inline-code">limit</code> and <code class="inline-code">ttl</code> (time to live). However, we added some other interesting ones, which we will look at in a bit.</p><p>We can read the first entry in the <code class="inline-code">options</code> array like so: The client is limited to 1 (limit) request per 1000ms (TTL), and if they exceed that, they will be blocked for 1000ms (blockDuration) during which the server responds with a 429. You can apply this model to read the remaining options.</p><p>While there are many other optional options (six others at the point of writing), we only used two: <code class="inline-code">blockDuration</code> and <code class="inline-code">name</code>. The former is quite intuitive; if not specified, it defaults to the TTL value. Note that time is always specified in milliseconds when using this module.</p><p>The <code class="inline-code">name</code> property defines a name for the group. If not specified, it resolves to the string <code class="inline-code">default</code>. This property helps during tracking to distinguish how tracking information will be stored across options.</p><p>We can also add options to define how we want to generate identifiers to track clients or whether, for some option groups, we may want to skip some routes in our app.</p><p>Now, for the root object, we also included another optional property called <code class="inline-code">errorMessage</code>, which accepts a string or a function that resolves to the message that will be sent when the user exceeds any of the defined limits.</p><p>Finally, we need to verify our entire application is guarded using the rate limiting options, so we used the special <code class="inline-code">APP_GUARD</code> injection token to provision the <code class="inline-code">ThrottlerGuard</code> class. The <code class="inline-code">ThrottlerGuard</code> class is where all the magic happens. It is responsible for tracking and maintaining the state for clients and blocking them when necessary.</p><p>To test our running application, open the terminal and start the server by running the following command:</p><pre class=" language-shell"><code class="prism  language-shell">npm run start:dev
</code></pre><p>Using a browser&rsquo;s address bar or any API testing platform, let&rsquo;s proceed to test our API. In this article, we will be using Progress Telerik <a target="_blank" href="https://www.telerik.com/fiddler/fiddler-everywhere">Fiddler Everywhere</a>. Notice that whenever we exceed any of the limits, we get a 429 response as shown below.</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-06/limits-is-exceeded.png?sfvrsn=158c49f1_2" title="429 response is returned If any of the limits is exceeded" alt="429 response is returned If any of the limits is exceeded" /></p><h2 id="how-tracking-works">How Tracking Works</h2><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-06/tracking-key-and-data.png?sfvrsn=1f80d533_2" title="Tracking using a tracking key and the tracking data" alt="Tracking using a tracking key and the tracking data" /></p><p>Tracking information is stored in a map using key-value pairs. A hyphen (-) separates the pieces that make up the tracking keys. The tracking key is derived from the name of the controller, the handling function, the name of the throttle option and a custom client identifier. By default, this is the client&rsquo;s IP address.</p><p>The diagram above shows the key in plain text, but internally they are hashed (SHA256) before they are used as keys.</p><p>The tracking data is derived from the throttle options. It maintains relevant information such as:</p><ul><li>How many requests the client made</li><li>The time for each limit</li><li>The block duration and status</li></ul><blockquote><p>Notice that there are three instances of the tracking keys and their corresponding tracking information maintained for each endpoint for each client, since we defined three different options.</p></blockquote><h2 id="overriding-rate-limiting-options-for-certain-routes">Overriding Rate Limiting Options for Certain Routes</h2><p>Update your <code class="inline-code">app.controller.ts</code> file to match the following:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> Controller<span class="token punctuation">,</span> Get <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> Throttle <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/throttler"</span><span class="token punctuation">;</span>
@<span class="token function">Controller</span><span class="token punctuation">(</span><span class="token string">"app"</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppController</span> <span class="token punctuation">{</span>
  @<span class="token function">Throttle</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
    first<span class="token punctuation">:</span> <span class="token punctuation">{</span>
      ttl<span class="token punctuation">:</span> <span class="token number">3000</span><span class="token punctuation">,</span>
      limit<span class="token punctuation">:</span> <span class="token number">1</span><span class="token punctuation">,</span>
      blockDuration<span class="token punctuation">:</span> <span class="token number">10000</span><span class="token punctuation">,</span>
    <span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token punctuation">}</span><span class="token punctuation">)</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/a"</span><span class="token punctuation">)</span>
  <span class="token function">getA</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is A"</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/b"</span><span class="token punctuation">)</span>
  <span class="token function">getB</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is B"</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/b"</span><span class="token punctuation">)</span>
  <span class="token function">getC</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is C"</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>Depending on what your application needs, it is often better to change the rate-limiting options either on one endpoint (i.e., <strong>at the route level</strong>) or a group of endpoints (i.e., <strong>at the controller level</strong>). Regardless of which, the <code class="inline-code">@nestjs/throttler</code> module provides us with the handy <code class="inline-code">Throttle</code> decorator to do so.</p><p>This function expects an object whose keys are the throttle option name attributes configured earlier in the <code class="inline-code">forRoot</code> method of the <code class="inline-code">ThrottlerModule</code>. The values are just overrides of the previous options.</p><p>In our case, we reconfigured the option called <code class="inline-code">first</code> and we limited it to one request every three seconds, with a block duration of 10 seconds.</p><p>Now if we try to hit <code class="inline-code">/app/a</code> more than once in three seconds, we get an error. Also note that <code class="inline-code">second</code> and <code class="inline-code">third</code> settings will still apply to this endpoint.</p><h2 id="partly-or-completely-disabling-rate-limiting-for-one-or-more-routes">Partly or Completely Disabling Rate Limiting for One or More Routes</h2><p>Not all parts of an application will need rate limiting. Update your <code class="inline-code">app.controller.ts</code> file with the following:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> Controller<span class="token punctuation">,</span> Get <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> SkipThrottle<span class="token punctuation">,</span> Throttle <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/throttler"</span><span class="token punctuation">;</span>
@<span class="token function">Controller</span><span class="token punctuation">(</span><span class="token string">"app"</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppController</span> <span class="token punctuation">{</span>
  <span class="token comment">//...</span>
  @<span class="token function">SkipThrottle</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
    first<span class="token punctuation">:</span> <span class="token keyword">true</span><span class="token punctuation">,</span>
    second<span class="token punctuation">:</span> <span class="token keyword">true</span><span class="token punctuation">,</span>
  <span class="token punctuation">}</span><span class="token punctuation">)</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/b"</span><span class="token punctuation">)</span>
  <span class="token function">getB</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is B"</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  <span class="token comment">//...</span>
<span class="token punctuation">}</span>
</code></pre><p>The <code class="inline-code">SkipThrottle</code> decorator allows us to disable rate limiting again, either on the controller or route level. It expects an object whose keys are the names of throttle options and whose values are booleans. For example, in our case we disabled <code class="inline-code">first</code> and <code class="inline-code">second</code> throttle options on the <code class="inline-code">/app/b</code> endpoint. This means only the rate limiting option named <code class="inline-code">third</code> will apply to this route.</p><h2 id="rate-limiting-problems-in-real-world-applications">Rate Limiting Problems in Real World Applications</h2><p>In this section, we will look at some common rate limiting problems we encounter in real-world applications and how we can solve them. Our solutions will use the <code class="inline-code">@nestjs/throttler</code> module and some of the features provided by the Nest framework:</p><ul><li>Tracking clients by IP address when our web server is behind a reverse proxy</li><li>Storing client tracking information across multiple instances</li></ul><h2 id="tracking-clients-by-ip-address-when-our-web-server-is-behind-a-reverse-proxy">Tracking Clients by IP Address When Our Web Server Is Behind a Reverse Proxy</h2><p>It has already been established that the <code class="inline-code">@nestjs/throttler</code> module allows us to track users based on different identifiers, but by default it does so using the client&rsquo;s IP, which is the most common option.</p><p>There are some interesting things to note here: When we deploy our server-side applications, in most cases, we don&rsquo;t allow direct client communication with them. Rather, we usually place our server behind another web server (e.g., Nginx, Apache, Caddy, Traefik, etc.) which serves as a load balancer and/or reverse proxy. Load balancers provide many benefits when paired with our web server, such as security from attackers, caching, compression, traffic distribution, etc.</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-06/forwards-client-request.png?sfvrsn=397f1c61_2" title="Proxy server forwards client request to NestJS web server" alt="Proxy server forwards client request to NestJS web server" /></p><p>Each time the client makes a request, it goes through the reverse proxy and then to our actual NestJS server. To verify our NestJS server knows about the client&rsquo;s IP where the request originated from, we need to check that the load balancer sends this information to it. When using a load balancer like Nginx, by default this information is not sent. This means our NestJS server treats the request as if it originated from Nginx, and this is not what we want.</p><p>For our demo let&rsquo;s create a docker-compose.yaml file in the project&rsquo;s root and add the following to it:</p><pre class=" language-yml"><code class="prism  language-yml"><span class="token key atrule">services</span><span class="token punctuation">:</span>
  <span class="token key atrule">backend1</span><span class="token punctuation">:</span>
    <span class="token key atrule">build</span><span class="token punctuation">:</span>
      <span class="token key atrule">context</span><span class="token punctuation">:</span> .
      <span class="token key atrule">dockerfile</span><span class="token punctuation">:</span> Dockerfile
      <span class="token key atrule">target</span><span class="token punctuation">:</span> development
    <span class="token key atrule">expose</span><span class="token punctuation">:</span>
      <span class="token comment"># used to expose port to other containers</span>
      <span class="token punctuation">-</span> <span class="token number">3000</span>
    <span class="token key atrule">volumes</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> ./src<span class="token punctuation">:</span>/app/src
      <span class="token punctuation">-</span> /app/node_modules
    <span class="token key atrule">command</span><span class="token punctuation">:</span> npm run start<span class="token punctuation">:</span>dev
    <span class="token key atrule">hostname</span><span class="token punctuation">:</span> backend1
    <span class="token key atrule">container_name</span><span class="token punctuation">:</span> backend1
  <span class="token key atrule">nginx</span><span class="token punctuation">:</span>
    <span class="token key atrule">image</span><span class="token punctuation">:</span> nginx
    <span class="token key atrule">depends_on</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> backend1
    <span class="token key atrule">ports</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token string">"8080:80"</span>
    <span class="token key atrule">volumes</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> ./nginx.conf<span class="token punctuation">:</span>/etc/nginx/nginx.conf
</code></pre><p>We set up two services: the first being our NestJS web server called <code class="inline-code">backend1</code>, which has a hostname of <code class="inline-code">backend1</code> and exposes port 3000 to other containers. The other is Nginx, the load balancer we will be using, that is exposed on our machine on <code class="inline-code">localhost:8080</code>. We created an <code class="inline-code">nginx.conf</code> file in our project root. We defined a bind mount to override the contents of the one that ships with the Nginx container.</p><p>This file looks like so:</p><pre class=" language-ts"><code class="prism  language-ts">worker_processes <span class="token number">1</span><span class="token punctuation">;</span>

events <span class="token punctuation">{</span>
    worker_connections <span class="token number">1024</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
http <span class="token punctuation">{</span>
    include mime<span class="token punctuation">.</span>types<span class="token punctuation">;</span>
    upstream backend_cluster <span class="token punctuation">{</span>
        # round robin algorithm is used by <span class="token keyword">default</span>
        server backend1<span class="token punctuation">:</span><span class="token number">3000</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    server <span class="token punctuation">{</span>
        listen <span class="token number">80</span><span class="token punctuation">;</span>
        server_name localhost<span class="token punctuation">;</span>
        location <span class="token operator">/</span> <span class="token punctuation">{</span>
            # Proxy requests to the upstream we called backend_cluster
            proxy_pass http<span class="token punctuation">:</span><span class="token operator">/</span><span class="token operator">/</span>backend_cluster<span class="token punctuation">;</span>
            # Pass the X<span class="token operator">-</span>Forwarded<span class="token operator">-</span>For header
            proxy_set_header X<span class="token operator">-</span>Forwarded<span class="token operator">-</span>For $proxy_add_x_forwarded_for<span class="token punctuation">;</span>
            proxy_set_header X<span class="token operator">-</span>Client<span class="token operator">-</span>Ip $remote_addr<span class="token punctuation">;</span>

            proxy_set_header Host $host<span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>We will focus on the more important blocks in this file:</p><p>The <code class="inline-code">proxy_pass</code> directive proxies requests from the load balancer to our backend cluster named <code class="inline-code">backend_cluster</code> in the upstream block. The upstream block for now has one entry that points to our NestJS server (i.e., &ldquo;backend1:3000&rdquo;).</p><p>To forward the client IP address, we used two custom headers to show two different ways we can do this. The first is a random header we called <code class="inline-code">X-Client-IP</code> and the other is the more popular <code class="inline-code">X-Forwarded-For</code> header, which we are more interested in. Both of them will set the client&rsquo;s IP, which is stored in the <code class="inline-code">$remote_addr</code> variable. <code class="inline-code">$proxy_add_x_forwarded_for</code> uses this internally and appends the client&rsquo;s IP to whatever was set in the <code class="inline-code">X-Forwarded-For</code> that came with the original request.</p><p>The Host header verifies that the upstream (i.e., our NestJS server) receives the right URL so that the right controller attends to the request.</p><h2 id="using-a-custom-header">Using a Custom Header</h2><p>If you prefer using the custom header, we can just directly extend the <code class="inline-code">ThrottlerGuard</code> and override its <code class="inline-code">getTracker()</code> method as shown below.</p><p>Let&rsquo;s update our <code class="inline-code">app.module.ts</code> file with the following:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token comment">// Add this</span>
@<span class="token function">Injectable</span><span class="token punctuation">(</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">ThrottlerBehindProxyGuard</span> <span class="token keyword">extends</span> <span class="token class-name">ThrottlerGuard</span> <span class="token punctuation">{</span>
  <span class="token keyword">protected</span> <span class="token keyword">async</span> <span class="token function">getTracker</span><span class="token punctuation">(</span>req<span class="token punctuation">:</span> Record<span class="token operator">&lt;</span><span class="token keyword">string</span><span class="token punctuation">,</span> <span class="token keyword">any</span><span class="token operator">&gt;</span><span class="token punctuation">)</span><span class="token punctuation">:</span> Promise<span class="token operator">&lt;</span><span class="token keyword">string</span><span class="token operator">&gt;</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> req<span class="token punctuation">.</span>headers<span class="token punctuation">[</span><span class="token string">"x-client-ip"</span><span class="token punctuation">]</span><span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>

@<span class="token function">Module</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
  imports<span class="token punctuation">:</span> <span class="token punctuation">[</span>
    ThrottlerModule<span class="token punctuation">.</span><span class="token function">forRoot</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
      throttlers<span class="token punctuation">:</span> <span class="token punctuation">[</span>
        <span class="token comment">//...throttlers</span>
      <span class="token punctuation">]</span><span class="token punctuation">,</span>
      errorMessage<span class="token punctuation">:</span> <span class="token string">"too many requests!"</span><span class="token punctuation">,</span>
    <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  controllers<span class="token punctuation">:</span> <span class="token punctuation">[</span>AppController<span class="token punctuation">]</span><span class="token punctuation">,</span>
  providers<span class="token punctuation">:</span> <span class="token punctuation">[</span>
    AppService<span class="token punctuation">,</span>
    <span class="token punctuation">{</span>
      provide<span class="token punctuation">:</span> APP_GUARD<span class="token punctuation">,</span>
      useClass<span class="token punctuation">:</span> ThrottlerBehindProxyGuard<span class="token punctuation">,</span>
    <span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
<span class="token punctuation">}</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppModule</span> <span class="token punctuation">{</span><span class="token punctuation">}</span>
</code></pre><p>In the code above, we retrieved the client&rsquo;s IP from the custom header set.</p><h2 id="using-the-x-forwarded-for-custom-header">Using the X-Forwarded-For Custom Header</h2><p>We can directly read the contents of the <code class="inline-code">X-Forwarded-For</code> header from the request object, like we did previously with our random <code class="inline-code">X-Client-IP</code> header. However, this header is a somewhat de facto header used by most proxy servers (e.g., Nginx in our case) to forward the client&rsquo;s IP to the web server behind it.</p><p>This header is already known to ExpressJS, so we just need to do a few things to enable the Express APIs to parse it for us automatically.</p><p>Let&rsquo;s now tweak our backend code to parse this value. Update your <code class="inline-code">main.ts</code> file with the following:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> NestFactory <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/core"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> AppModule <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"./app.module"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> NestExpressApplication <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/platform-express"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> Injectable<span class="token punctuation">,</span> ExecutionContext<span class="token punctuation">,</span> CallHandler <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>

<span class="token keyword">async</span> <span class="token keyword">function</span> <span class="token function">bootstrap</span><span class="token punctuation">(</span><span class="token punctuation">)</span> <span class="token punctuation">{</span>
  <span class="token keyword">const</span> app <span class="token operator">=</span> <span class="token keyword">await</span> NestFactory<span class="token punctuation">.</span>create<span class="token operator">&lt;</span>NestExpressApplication<span class="token operator">&gt;</span><span class="token punctuation">(</span>AppModule<span class="token punctuation">)</span><span class="token punctuation">;</span>
  app<span class="token punctuation">.</span><span class="token keyword">set</span><span class="token punctuation">(</span><span class="token string">"trust proxy"</span><span class="token punctuation">,</span> <span class="token keyword">true</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
  <span class="token keyword">await</span> app<span class="token punctuation">.</span><span class="token function">listen</span><span class="token punctuation">(</span><span class="token number">3000</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
<span class="token punctuation">}</span>
<span class="token function">bootstrap</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre><p>NestJS allows us to choose between Express or Fastify, and we are going to use the former.</p><p>By setting the <code class="inline-code">trust proxy</code> setting on the Express API, it automatically tells Express to trust the <code class="inline-code">X-Forwarded-For</code> header and parse it to an array, storing it in the <code class="inline-code">request</code> object in its ips array (i.e., req.ips).</p><p>It then picks the leftmost entry in this array and stores it in <code class="inline-code">req.ip</code>. To put it in more context: assuming the <code class="inline-code">X-Forwarded-For</code> holds &ldquo;a b c&rdquo; (assuming there were two hops before the request reached our web server, where a, b and c are IP addresses), <code class="inline-code">req.ips</code> will hold [a, b, c] and <code class="inline-code">req.ip</code> holds <code class="inline-code">a</code>, where <code class="inline-code">a</code> is the client&rsquo;s IP address.</p><p>By default, since the <code class="inline-code">@nestjs/throttler</code> module relies on the <code class="inline-code">req.ip</code> property for tracking, everything works automatically just by using the <code class="inline-code">trust proxy</code> setting without having to extend the <code class="inline-code">ThrottlerGuard</code> class like we did to manually extract the contents of our <code class="inline-code">X-Client-IP</code> header for <code class="inline-code">@nestjs/throttler</code>.</p><h2 id="storing-client-tracking-information-across-multiple-instances">Storing Client Tracking Information Across Multiple Instances</h2><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-06/multiple-instances.png?sfvrsn=7aa5093c_2" title="Multiple instances of a web server behind load balancer" alt="Multiple instances of a web server behind load balancer" /></p><p>Another great thing about having our web server behind a load balancer is that it allows us to run multiple instances of the web server while the load balancer distributes incoming traffic across all instances. This is important for throughput, availability and proper resource utilization.</p><p>In the context of rate limiting, this also presents some storage concerns. First, by default, since the <code class="inline-code">@nestjs/throttler</code> module uses an in-memory store (a JavaScript Map data structure, to be specific), to remember tracking information for rate limiting, this means when we have multiple instances of our app, each instance maintains its own in-memory store, as shown below.</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-06/instance-maintains-memory.png?sfvrsn=d925e60b_2" title="Each instance maintains its own in memory storage" alt="Each instance maintains its own in memory storage" /></p><p>This storage decentralization presents a problem (i.e., assuming we have three instances, a, b and c, it is possible for a client to be blocked on one of the servers, e.g., a, and still be able to access b and c).</p><p>So we need to centralize the storage location for the tracking information and move to a structure that looks like the one below.</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-06/storing-tracking-information.png?sfvrsn=c64fe7a4_2" title="Storing tracking information in a centralized storage" alt="Storing tracking information in a centralized storage" /></p><p>The <code class="inline-code">@nestjs/throttler</code> module allows us to provide our storage option if we want, provided it implements the <code class="inline-code">ThrottlerStorage</code> interface. We won&rsquo;t be creating ours; instead, we will use a community-provided package that allows us to use Redis as the persistence layer.</p><p>Let&rsquo;s install this module in our app. Run the following command in your terminal:</p><pre class=" language-shell"><code class="prism  language-shell">npm install --save @nest-lab/throttler-storage-redis ioredis
</code></pre><p>Next, let us update our <code class="inline-code">docker-compose.yaml</code> file to set up Redis, and then make three instances of our web server in our file:</p><pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">services</span><span class="token punctuation">:</span>
  <span class="token key atrule">backend1</span><span class="token punctuation">:</span>
    <span class="token key atrule">build</span><span class="token punctuation">:</span>
      <span class="token key atrule">context</span><span class="token punctuation">:</span> .
      <span class="token key atrule">dockerfile</span><span class="token punctuation">:</span> Dockerfile
      <span class="token key atrule">target</span><span class="token punctuation">:</span> development
    <span class="token key atrule">ports</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token string">"3002:3000"</span>
    <span class="token key atrule">expose</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token number">3000</span>
    <span class="token key atrule">environment</span><span class="token punctuation">:</span>
      <span class="token key atrule">INSTANCE_NAME</span><span class="token punctuation">:</span> BACKEND_1
    <span class="token key atrule">volumes</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> ./src<span class="token punctuation">:</span>/app/src
      <span class="token punctuation">-</span> /app/node_modules
    <span class="token key atrule">command</span><span class="token punctuation">:</span> npm run start<span class="token punctuation">:</span>dev
    <span class="token key atrule">hostname</span><span class="token punctuation">:</span> backend1
    <span class="token key atrule">container_name</span><span class="token punctuation">:</span> backend1
  <span class="token key atrule">backend2</span><span class="token punctuation">:</span>
    <span class="token key atrule">build</span><span class="token punctuation">:</span>
      <span class="token key atrule">context</span><span class="token punctuation">:</span> .
      <span class="token key atrule">dockerfile</span><span class="token punctuation">:</span> Dockerfile
      <span class="token key atrule">target</span><span class="token punctuation">:</span> development
    <span class="token key atrule">ports</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token string">"3003:3000"</span>
    <span class="token key atrule">expose</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token number">3000</span>
    <span class="token key atrule">environment</span><span class="token punctuation">:</span>
      <span class="token key atrule">INSTANCE_NAME</span><span class="token punctuation">:</span> BACKEND_2
    <span class="token key atrule">volumes</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> ./src<span class="token punctuation">:</span>/app/src
      <span class="token punctuation">-</span> /app/node_modules
    <span class="token key atrule">command</span><span class="token punctuation">:</span> npm run start<span class="token punctuation">:</span>dev
    <span class="token key atrule">hostname</span><span class="token punctuation">:</span> backend2
    <span class="token key atrule">container_name</span><span class="token punctuation">:</span> backend2
  <span class="token key atrule">backend3</span><span class="token punctuation">:</span>
    <span class="token key atrule">build</span><span class="token punctuation">:</span>
      <span class="token key atrule">context</span><span class="token punctuation">:</span> .
      <span class="token key atrule">dockerfile</span><span class="token punctuation">:</span> Dockerfile
      <span class="token key atrule">target</span><span class="token punctuation">:</span> development
    <span class="token key atrule">ports</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token string">"3004:3004"</span>
    <span class="token key atrule">expose</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token number">3000</span>
    <span class="token key atrule">environment</span><span class="token punctuation">:</span>
      <span class="token key atrule">INSTANCE_NAME</span><span class="token punctuation">:</span> BACKEND_3
    <span class="token key atrule">volumes</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> ./src<span class="token punctuation">:</span>/app/src
      <span class="token punctuation">-</span> /app/node_modules
    <span class="token key atrule">command</span><span class="token punctuation">:</span> npm run start<span class="token punctuation">:</span>dev
    <span class="token key atrule">hostname</span><span class="token punctuation">:</span> backend3
    <span class="token key atrule">container_name</span><span class="token punctuation">:</span> backend3
  <span class="token key atrule">nginx</span><span class="token punctuation">:</span>
    <span class="token comment">#... nginx service</span>
  <span class="token key atrule">redis</span><span class="token punctuation">:</span>
    <span class="token key atrule">image</span><span class="token punctuation">:</span> redis/redis<span class="token punctuation">-</span>stack
    <span class="token key atrule">environment</span><span class="token punctuation">:</span>
      <span class="token key atrule">REDIS_ARGS</span><span class="token punctuation">:</span> <span class="token string">"--requirepass 1234"</span>
      <span class="token key atrule">REDISSEARCH_ARGS</span><span class="token punctuation">:</span> <span class="token string">"--requirepass 1234"</span>
      <span class="token key atrule">REDIS_HOST</span><span class="token punctuation">:</span> redis
      <span class="token key atrule">REDIS_USERNAME</span><span class="token punctuation">:</span> default
      <span class="token key atrule">REDIS_PASSWORD</span><span class="token punctuation">:</span> <span class="token number">1234</span>
      <span class="token key atrule">REDIS_DATABASE_NAME</span><span class="token punctuation">:</span> <span class="token number">0</span>
      <span class="token key atrule">REDIS_PORT</span><span class="token punctuation">:</span> <span class="token number">8001</span>
    <span class="token key atrule">volumes</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> ./local<span class="token punctuation">-</span>data/<span class="token punctuation">:</span>/data
    <span class="token key atrule">ports</span><span class="token punctuation">:</span>
      <span class="token punctuation">-</span> <span class="token string">"6379:6379"</span>
      <span class="token punctuation">-</span> <span class="token string">"8001:8001"</span>
</code></pre><p>Notice we included an environment variable for each instance called <code class="inline-code">INSTANCE_NAME</code>. This will be used later so we know which instance of our web server is handling the proxied request.</p><p>Let&rsquo;s update our upstream block in our <code class="inline-code">nginx.conf</code> file to include the two other server instances, making them 3:</p><pre class=" language-ts"><code class="prism  language-ts">worker_processes <span class="token number">1</span><span class="token punctuation">;</span>

events <span class="token punctuation">{</span>
    worker_connections <span class="token number">1024</span><span class="token punctuation">;</span>
    #
<span class="token punctuation">}</span>
http <span class="token punctuation">{</span>
    # ensures that the server includes the proper mimetypes when the response is sent back to the client
    include mime<span class="token punctuation">.</span>types<span class="token punctuation">;</span>
    upstream backend_cluster <span class="token punctuation">{</span>
        # Define the backend servers using round<span class="token operator">-</span>robin
        server backend1<span class="token punctuation">:</span><span class="token number">3000</span><span class="token punctuation">;</span>
        server backend2<span class="token punctuation">:</span><span class="token number">3000</span><span class="token punctuation">;</span>
        server backend3<span class="token punctuation">:</span><span class="token number">3000</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
    server <span class="token punctuation">{</span>
        # server block
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>Before we start our fleet of services, we need to do a few things. Let&rsquo;s start by updating the <code class="inline-code">app.module.ts file</code> to use the Redis storage module:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> Module <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> AppService <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"./app.service"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span>
  minutes<span class="token punctuation">,</span>
  seconds<span class="token punctuation">,</span>
  ThrottlerGuard<span class="token punctuation">,</span>
  ThrottlerModule<span class="token punctuation">,</span>
<span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/throttler"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> APP_GUARD <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/core"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> AppController <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"./app.controller"</span><span class="token punctuation">;</span>
<span class="token comment">// throttler-behind-proxy.guard.ts</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> Injectable <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> ThrottlerStorageRedisService <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nest-lab/throttler-storage-redis"</span><span class="token punctuation">;</span>

@<span class="token function">Module</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
  imports<span class="token punctuation">:</span> <span class="token punctuation">[</span>
    ThrottlerModule<span class="token punctuation">.</span><span class="token function">forRoot</span><span class="token punctuation">(</span><span class="token punctuation">{</span>
      throttlers<span class="token punctuation">:</span> <span class="token punctuation">[</span>
        <span class="token comment">// {</span>
        <span class="token comment">//   name: 'first',</span>
        <span class="token comment">//   ttl: 1000,</span>
        <span class="token comment">//   limit: 1,</span>
        <span class="token comment">//   blockDuration: 1000,</span>
        <span class="token comment">// },</span>
        <span class="token punctuation">{</span>
          name<span class="token punctuation">:</span> <span class="token string">"second"</span><span class="token punctuation">,</span>
          ttl<span class="token punctuation">:</span> <span class="token function">seconds</span><span class="token punctuation">(</span><span class="token number">10</span><span class="token punctuation">)</span><span class="token punctuation">,</span> <span class="token comment">// 10000</span>
          limit<span class="token punctuation">:</span> <span class="token number">4</span><span class="token punctuation">,</span>
        <span class="token punctuation">}</span><span class="token punctuation">,</span>
        <span class="token comment">// {</span>
        <span class="token comment">//   name: 'third',</span>
        <span class="token comment">//   ttl: minutes(1), // 60000</span>
        <span class="token comment">//   limit: 25,</span>
        <span class="token comment">// },</span>
      <span class="token punctuation">]</span><span class="token punctuation">,</span>
      errorMessage<span class="token punctuation">:</span> <span class="token string">"too many requests! on "</span> <span class="token operator">+</span> process<span class="token punctuation">.</span>env<span class="token punctuation">.</span>INSTANCE_NAME<span class="token punctuation">,</span>
      storage<span class="token punctuation">:</span> <span class="token keyword">new</span> <span class="token class-name">ThrottlerStorageRedisService</span><span class="token punctuation">(</span>
        <span class="token template-string"><span class="token string">`redis://default:1234@host.docker.internal:6379/0`</span></span>
      <span class="token punctuation">)</span><span class="token punctuation">,</span>
    <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
  controllers<span class="token punctuation">:</span> <span class="token punctuation">[</span>AppController<span class="token punctuation">]</span><span class="token punctuation">,</span>
  providers<span class="token punctuation">:</span> <span class="token punctuation">[</span>
    AppService<span class="token punctuation">,</span>
    <span class="token punctuation">{</span>
      provide<span class="token punctuation">:</span> APP_GUARD<span class="token punctuation">,</span>
      useClass<span class="token punctuation">:</span> ThrottlerGuard<span class="token punctuation">,</span>
    <span class="token punctuation">}</span><span class="token punctuation">,</span>
  <span class="token punctuation">]</span><span class="token punctuation">,</span>
<span class="token punctuation">}</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppModule</span> <span class="token punctuation">{</span><span class="token punctuation">}</span>
</code></pre><p>We included a connection string to connect to Redis. The username and password fields are hardcoded here; they match those specified in the REDIS_ARGS: <code class="inline-code">--requirepass 1234</code> and the <code class="inline-code">REDIS_USERNAME</code> default in our <code class="inline-code">docker-compose.yaml</code> file.</p><p>Also, we temporarily disabled all the other throttle options except the one named <code class="inline-code">second</code>. For now, the user is limited to four requests every 10 seconds and will be blocked for 10 seconds if they exceed the limit.</p><p>We also modified the error message to include the <code class="inline-code">INSTANCE_NAME</code> environment variable to know which instance the client is blocked on.</p><p>So that we know which server responds when we hit each endpoint, let&rsquo;s now update our <code class="inline-code">app.controller.ts</code> file:</p><pre class=" language-ts"><code class="prism  language-ts"><span class="token keyword">import</span> <span class="token punctuation">{</span> Controller<span class="token punctuation">,</span> Get <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/common"</span><span class="token punctuation">;</span>
<span class="token keyword">import</span> <span class="token punctuation">{</span> SkipThrottle<span class="token punctuation">,</span> Throttle <span class="token punctuation">}</span> <span class="token keyword">from</span> <span class="token string">"@nestjs/throttler"</span><span class="token punctuation">;</span>
@<span class="token function">Controller</span><span class="token punctuation">(</span><span class="token string">"app"</span><span class="token punctuation">)</span>
<span class="token keyword">export</span> <span class="token keyword">class</span> <span class="token class-name">AppController</span> <span class="token punctuation">{</span>
  <span class="token comment">// @Throttle({</span>
  <span class="token comment">//   first: {</span>
  <span class="token comment">//     ttl: 3000,</span>
  <span class="token comment">//     limit: 1,</span>
  <span class="token comment">//     blockDuration: 10000,</span>
  <span class="token comment">//   },</span>
  <span class="token comment">// })</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/a"</span><span class="token punctuation">)</span>
  <span class="token function">getA</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is A on "</span> <span class="token operator">+</span> process<span class="token punctuation">.</span>env<span class="token punctuation">.</span>INSTANCE_NAME<span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  <span class="token comment">// @SkipThrottle({</span>
  <span class="token comment">//   first: true,</span>
  <span class="token comment">//   second: true,</span>
  <span class="token comment">// })</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/b"</span><span class="token punctuation">)</span>
  <span class="token function">getB</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is B on "</span> <span class="token operator">+</span> process<span class="token punctuation">.</span>env<span class="token punctuation">.</span>INSTANCE_NAME<span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
  @<span class="token function">Get</span><span class="token punctuation">(</span><span class="token string">"/b"</span><span class="token punctuation">)</span>
  <span class="token function">getC</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">:</span> <span class="token keyword">string</span> <span class="token punctuation">{</span>
    <span class="token keyword">return</span> <span class="token string">"this is C on "</span> <span class="token operator">+</span> process<span class="token punctuation">.</span>env<span class="token punctuation">.</span>INSTANCE_NAME<span class="token punctuation">;</span>
  <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>We appended the <code class="inline-code">INSTANCE_NAME</code> environment variable to each endpoint response string to know which of our backend instances is responding to our request.</p><p>Let&rsquo;s rebuild our images and restart our fleet of services again by running:</p><pre class=" language-shell"><code class="prism  language-shell">docker compose up -d --build
</code></pre><p>Now, when we hit our reverse proxy at localhost:8080/app/a:</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-06/blocked-instances-centralized-storage.gif?sfvrsn=6d9dc9d1_2" title="Once client exceeds limit, they are blocked across all instances because of centralized storage" alt="Once client exceeds limit, they are blocked across all instances because of centralized storage" /></p><p>Based on our throttle options definition, we defined limits that restrict the client to four requests every 10 seconds, and will be blocked for 10 seconds if they exceed the limit. Notice in the image above that when the limit is exceeded, the client is blocked across all instances.</p><h2 id="conclusion">Conclusion</h2><p>Rate limiting helps force clients to consume resources responsibly. This guide shows us how we can use this technique to secure our web server and its resources from abuse.</p><aside><hr data-sf-ec-immutable="" /><div class="row"><div class="col-4 u-normal-full u-small-mb0"><h4 class="u-fs20 u-fw5 u-lh125 u-mb0">A Guide to Building GraphQL APIs with NestJS</h4></div><div class="col-8"><p class="u-fs16 u-mb0">Learn how GraphQL works, <a target="_blank" href="https://www.telerik.com/blogs/guide-building-graphql-apis-nestjs">how to use GraphQL with NestJS</a>, and how to use TypeORM with NestJS for our Postgres database.</p></div></div></aside><img src="https://feeds.telerik.com/link/23069/17200233.gif" height="1" width="1"/>
