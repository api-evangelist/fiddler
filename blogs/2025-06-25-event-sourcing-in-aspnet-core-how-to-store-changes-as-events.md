---
title: "Event Sourcing in ASP.NET Core: How to Store Changes as Events"
url: "https://feeds.telerik.com/link/23069/17063183/event-sourcing-aspnet-core-how-store-changes-events"
date: "2025-06-25T09:51:09Z"
author: "Assis Zang"
feed_url: "https://feeds.telerik.com/blogs/productivity-debugging"
---
<p><span class="featured">Learn how to implement the Event Sourcing architectural pattern in your ASP.NET Core app to record event-related system changes.</span></p><p>Have you ever imagined having a complete history of data in a distributed application? With Event Sourcing, each change becomes an immutable event, for full traceability in ASP.NET Core.</p><p>Event Sourcing is an architectural pattern widely used in large and medium-sized systems due to its versatility in dealing with complexities through an approach different from traditional approaches: Instead of saving the current state, it records each change in the system as an event.</p><p>In this post, we will understand how Event Sourcing works and what its advantages are compared to traditional approaches. We will also implement an ASP.NET Core application using the principles of Event Sourcing.</p><h2 id="understanding-event-sourcing">Understanding Event Sourcing</h2><p>Event Sourcing is an architectural pattern that aims to handle changes through immutable events, storing the complete history of all changes made to a system, allowing the reconstruction of the state at any time.</p><p>Imagine a bank account management platform. Instead of having an Account entity, which has a balance that is updated with each new transaction, in Event Sourcing, each transaction (deposit, withdrawal, transfer) is recorded as an event that does not change. Each event is recorded exactly as it was sent, so the current account balance is derived from the amounts deposited and withdrawn in previous events. This allows tracking the entire history of account movements, so all information from the first change to the last is present.</p><p>The image below demonstrates how movements in a bank account are handled using the traditional and Event Sourcing approaches.</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-04/event-sourcing-vs-traditional-approach.png?sfvrsn=a274f7d0_2" title="event-sourcing vs traditional approach" alt="Event Sourcing  VS Traditional Approach" /></p><h2 id="advantages-of-using-events">Advantages of Using Events</h2><p>The use of events brings several advantages, especially in systems that need to maintain traceability (history) and be scalable and resilient. Below are some of the main aspects that make the use of events a great choice:</p><p><strong>Complete data history</strong>: The use of events allows you to have an immutable history of everything that happened. This facilitates audits, debugging and analysis of the behavior of the system during its life cycle.</p><p><strong>Reproduction and reconstruction of the state</strong>: It is possible to reconstruct the current state of any entity, simply by reapplying past events.</p><p><strong>Scalability and performance</strong>: The use of events allows the separation between writing (events) and reading (projections), allowing optimizations for each one in a specific way. In addition, systems can be designed to scale horizontally, since events are stored asynchronously.</p><p><strong>Easy asynchronous integration</strong>: Allows the creation of decoupled microservices, where services react to events without the need for direct calls.</p><p><strong>Resilience and fault tolerance</strong>: In case of failure, simply reprocess the events to restore the system state. In addition, there is no risk of data loss, as events are stored immutably and are always available.</p><h2 id="implementing-event-sourcing-in-asp.net-core">Implementing Event Sourcing in ASP.NET Core</h2><p>Now we will implement an ASP.NET Core application that uses the Event Sourcing pattern to send and receive deposit and withdrawal events. For this, we will use two libraries: RabbitMQ and MassTransit.</p><p>RabbitMQ is a message broker that implements the Advanced Message Queuing Protocol (AMQP). It acts as an intermediary between message producers and consumers, allowing them to communicate in a decoupled and asynchronous way.</p><p>MassTransit is a .NET library for asynchronous messaging. It simplifies integration with message brokers like RabbitMQ or Azure Service by abstracting away complex details and supporting messaging patterns like publish/subscribe, request/response and saga orchestration.</p><h3 id="prerequisites">Prerequisites</h3><p>Below are some prerequisites that you need to meet to reproduce the tutorial in the post.</p><ol><li>.NET version &ndash; You will need the latest version of .NET (.NET 9 or higher)</li><li>Docker &ndash; Required to run the local RabbitMQ service</li></ol><h2 id="creating-the-application-and-installing-the-dependencies">Creating the Application and Installing the Dependencies</h2><p>To create the application, you can use the command below:</p><pre class=" language-bash"><code class="prism  language-bash">dotnet new web -o BankStream
</code></pre><p>Then, in the project root, you can use the following commands to install the NuGet packages:</p><pre class=" language-bash"><code class="prism  language-bash">dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.AspNetCore
</code></pre><h2 id="creating-the-events">Creating the Events</h2><p>Now let&rsquo;s create the entities that represent the deposit and withdrawal events. They will be simple, as the focus is to demonstrate the sending and consumption of the events.</p><p>So, create a new folder called &ldquo;Events&rdquo; and, inside it, create the following records:</p><ul><li>DepositEvent</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Events<span class="token punctuation">;</span>

<span class="token keyword">public</span> record <span class="token function">DepositEvent</span><span class="token punctuation">(</span>Guid Id<span class="token punctuation">,</span> <span class="token keyword">string</span> AccountId<span class="token punctuation">,</span> <span class="token keyword">decimal</span> Amount<span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre><ul><li>WithdrawalEvent</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Events<span class="token punctuation">;</span>

<span class="token keyword">public</span> record <span class="token function">WithdrawalEvent</span><span class="token punctuation">(</span>Guid Id<span class="token punctuation">,</span> <span class="token keyword">string</span> AccountId<span class="token punctuation">,</span> <span class="token keyword">decimal</span> Amount<span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre><h2 id="creating-the-consumers">Creating the Consumers</h2><p>Then, let&rsquo;s create the consumers. They are responsible for receiving the events and processing the actions related to it. In this case, update the status and register it in the database.</p><p>First, let&rsquo;s create the classes and enums used by the consumers. So, create a new folder called &ldquo;Models&rdquo; and, inside it, create the following classes:</p><ul><li>StatusEnum</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Models<span class="token punctuation">;</span>
<span class="token keyword">public</span> <span class="token keyword">enum</span> StatusEnum
<span class="token punctuation">{</span>
    Pending<span class="token punctuation">,</span>
    Completed<span class="token punctuation">,</span>
    Failed<span class="token punctuation">,</span>
    Canceled<span class="token punctuation">,</span>
    Reversed<span class="token punctuation">,</span>
    InProgress<span class="token punctuation">,</span>
    OnHold
<span class="token punctuation">}</span>
</code></pre><ul><li>TransactionTypeEnum</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Models<span class="token punctuation">;</span>

<span class="token keyword">public</span> <span class="token keyword">enum</span> TransactionTypeEnum
<span class="token punctuation">{</span>
    Deposit<span class="token punctuation">,</span>
    Withdrawal
<span class="token punctuation">}</span>
</code></pre><ul><li>TransactionStatus</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Models<span class="token punctuation">;</span>

<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">TransactionStatus</span>
<span class="token punctuation">{</span>
    <span class="token keyword">public</span> <span class="token function">TransactionStatus</span><span class="token punctuation">(</span><span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
    <span class="token punctuation">}</span>

    <span class="token keyword">public</span> <span class="token function">TransactionStatus</span><span class="token punctuation">(</span>Guid id<span class="token punctuation">,</span> <span class="token keyword">string</span> accountId<span class="token punctuation">,</span> StatusEnum statusEnum<span class="token punctuation">,</span> <span class="token keyword">decimal</span> amount<span class="token punctuation">,</span> <span class="token keyword">bool</span> isSuccess<span class="token punctuation">,</span> <span class="token keyword">string</span><span class="token operator">?</span> errorMessage<span class="token punctuation">,</span> TransactionTypeEnum type<span class="token punctuation">,</span> DateTime createdAt<span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
        Id <span class="token operator">=</span> id<span class="token punctuation">;</span>
        AccountId <span class="token operator">=</span> accountId<span class="token punctuation">;</span>
        StatusEnum <span class="token operator">=</span> statusEnum<span class="token punctuation">;</span>
        Amount <span class="token operator">=</span> amount<span class="token punctuation">;</span>
        IsSuccess <span class="token operator">=</span> isSuccess<span class="token punctuation">;</span>
        ErrorMessage <span class="token operator">=</span> errorMessage<span class="token punctuation">;</span>
        Type <span class="token operator">=</span> type<span class="token punctuation">;</span>
        CreatedAt <span class="token operator">=</span> createdAt<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>

    <span class="token keyword">public</span> Guid Id <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> <span class="token keyword">string</span> AccountId <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> StatusEnum StatusEnum <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span><span class="token punctuation">}</span>
    <span class="token keyword">public</span> <span class="token keyword">decimal</span> Amount <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> <span class="token keyword">bool</span> IsSuccess <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> <span class="token keyword">string</span><span class="token operator">?</span> ErrorMessage <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> TransactionTypeEnum Type <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> DateTime CreatedAt <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span> <span class="token operator">=</span> DateTime<span class="token punctuation">.</span>UtcNow<span class="token punctuation">;</span>
<span class="token punctuation">}</span>
</code></pre><p>Then, create a new folder called &ldquo;Data&rdquo; and inside it add the class below:</p><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Events<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Models<span class="token punctuation">;</span>
<span class="token keyword">using</span> Microsoft<span class="token punctuation">.</span>EntityFrameworkCore<span class="token punctuation">;</span>

<span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Data<span class="token punctuation">;</span>

<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">EventDbContext</span> <span class="token punctuation">:</span> DbContext
<span class="token punctuation">{</span>
    <span class="token keyword">public</span> DbSet<span class="token operator">&lt;</span>DepositEvent<span class="token operator">&gt;</span> Deposits <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> DbSet<span class="token operator">&lt;</span>WithdrawalEvent<span class="token operator">&gt;</span> Withdrawals <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>
    <span class="token keyword">public</span> DbSet<span class="token operator">&lt;</span>TransactionStatus<span class="token operator">&gt;</span> TransactionStatus <span class="token punctuation">{</span> <span class="token keyword">get</span><span class="token punctuation">;</span> <span class="token keyword">set</span><span class="token punctuation">;</span> <span class="token punctuation">}</span>

    <span class="token keyword">public</span> <span class="token function">EventDbContext</span><span class="token punctuation">(</span>DbContextOptions<span class="token operator">&lt;</span>EventDbContext<span class="token operator">&gt;</span> options<span class="token punctuation">)</span>
        <span class="token punctuation">:</span> <span class="token keyword">base</span><span class="token punctuation">(</span>options<span class="token punctuation">)</span> <span class="token punctuation">{</span> <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>Now, let&rsquo;s create the consumer classes. Create a new folder called &ldquo;Consumers&rdquo; and add to it the following classes:</p><ul><li>DepositConsumer</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Data<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Events<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Models<span class="token punctuation">;</span>
<span class="token keyword">using</span> MassTransit<span class="token punctuation">;</span>

<span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Consumers<span class="token punctuation">;</span>

<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">DepositConsumer</span> <span class="token punctuation">:</span> IConsumer<span class="token operator">&lt;</span>DepositEvent<span class="token operator">&gt;</span>
<span class="token punctuation">{</span>
    <span class="token keyword">private</span> <span class="token keyword">readonly</span> EventDbContext _dbContext<span class="token punctuation">;</span>

    <span class="token keyword">public</span> <span class="token function">DepositConsumer</span><span class="token punctuation">(</span>EventDbContext dbContext<span class="token punctuation">)</span> <span class="token operator">=</span><span class="token operator">&gt;</span> _dbContext <span class="token operator">=</span> dbContext<span class="token punctuation">;</span>

    <span class="token keyword">public</span> <span class="token keyword">async</span> Task <span class="token function">Consume</span><span class="token punctuation">(</span>ConsumeContext<span class="token operator">&lt;</span>DepositEvent<span class="token operator">&gt;</span> context<span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
        <span class="token keyword">var</span> deposit <span class="token operator">=</span> context<span class="token punctuation">.</span>Message<span class="token punctuation">;</span>

        <span class="token keyword">try</span>
        <span class="token punctuation">{</span>
            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>TransactionStatus<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">TransactionStatus</span><span class="token punctuation">(</span>
                Guid<span class="token punctuation">.</span><span class="token function">NewGuid</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
                deposit<span class="token punctuation">.</span>AccountId<span class="token punctuation">,</span>
                StatusEnum<span class="token punctuation">.</span>Completed<span class="token punctuation">,</span>
                deposit<span class="token punctuation">.</span>Amount<span class="token punctuation">,</span>
                <span class="token keyword">true</span><span class="token punctuation">,</span>
                <span class="token keyword">string</span><span class="token punctuation">.</span>Empty<span class="token punctuation">,</span>
                TransactionTypeEnum<span class="token punctuation">.</span>Deposit<span class="token punctuation">,</span>
                DateTime<span class="token punctuation">.</span>Now<span class="token punctuation">)</span>
            <span class="token punctuation">)</span><span class="token punctuation">;</span>

            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>Deposits<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span>deposit<span class="token punctuation">)</span><span class="token punctuation">;</span>

            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span><span class="token function">SaveChangesAsync</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">Exception</span> ex<span class="token punctuation">)</span>
        <span class="token punctuation">{</span>
            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>TransactionStatus<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">TransactionStatus</span><span class="token punctuation">(</span>
                Guid<span class="token punctuation">.</span><span class="token function">NewGuid</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
                deposit<span class="token punctuation">.</span>AccountId<span class="token punctuation">,</span>
                StatusEnum<span class="token punctuation">.</span>Failed<span class="token punctuation">,</span>
                deposit<span class="token punctuation">.</span>Amount<span class="token punctuation">,</span>
                <span class="token keyword">true</span><span class="token punctuation">,</span>
                ex<span class="token punctuation">.</span>Message<span class="token punctuation">,</span>
                TransactionTypeEnum<span class="token punctuation">.</span>Deposit<span class="token punctuation">,</span>
                DateTime<span class="token punctuation">.</span>Now<span class="token punctuation">)</span>
            <span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span><span class="token function">SaveChangesAsync</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">throw</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><ul><li>WithdrawalConsumer</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Data<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Events<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Models<span class="token punctuation">;</span>
<span class="token keyword">using</span> MassTransit<span class="token punctuation">;</span>

<span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Consumers<span class="token punctuation">;</span>

<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">WithdrawalConsumer</span> <span class="token punctuation">:</span> IConsumer<span class="token operator">&lt;</span>WithdrawalEvent<span class="token operator">&gt;</span>
<span class="token punctuation">{</span>
    <span class="token keyword">private</span> <span class="token keyword">readonly</span> EventDbContext _dbContext<span class="token punctuation">;</span>

    <span class="token keyword">public</span> <span class="token function">WithdrawalConsumer</span><span class="token punctuation">(</span>EventDbContext dbContext<span class="token punctuation">)</span> <span class="token operator">=</span><span class="token operator">&gt;</span> _dbContext <span class="token operator">=</span> dbContext<span class="token punctuation">;</span>

    <span class="token keyword">public</span> <span class="token keyword">async</span> Task <span class="token function">Consume</span><span class="token punctuation">(</span>ConsumeContext<span class="token operator">&lt;</span>WithdrawalEvent<span class="token operator">&gt;</span> context<span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
        <span class="token keyword">var</span> withdrawal <span class="token operator">=</span> context<span class="token punctuation">.</span>Message<span class="token punctuation">;</span>
        <span class="token keyword">try</span>
        <span class="token punctuation">{</span>
            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>TransactionStatus<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">TransactionStatus</span><span class="token punctuation">(</span>
                Guid<span class="token punctuation">.</span><span class="token function">NewGuid</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
                withdrawal<span class="token punctuation">.</span>AccountId<span class="token punctuation">,</span>
                StatusEnum<span class="token punctuation">.</span>Completed<span class="token punctuation">,</span>
                withdrawal<span class="token punctuation">.</span>Amount<span class="token punctuation">,</span>
                <span class="token keyword">true</span><span class="token punctuation">,</span>
                <span class="token keyword">string</span><span class="token punctuation">.</span>Empty<span class="token punctuation">,</span>
                TransactionTypeEnum<span class="token punctuation">.</span>Withdrawal<span class="token punctuation">,</span>
                DateTime<span class="token punctuation">.</span>Now<span class="token punctuation">)</span>
            <span class="token punctuation">)</span><span class="token punctuation">;</span>

            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>Withdrawals<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span>context<span class="token punctuation">.</span>Message<span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span><span class="token function">SaveChangesAsync</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
        <span class="token keyword">catch</span> <span class="token punctuation">(</span><span class="token class-name">Exception</span> ex<span class="token punctuation">)</span>
        <span class="token punctuation">{</span>
            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>TransactionStatus<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">TransactionStatus</span><span class="token punctuation">(</span>
                Guid<span class="token punctuation">.</span><span class="token function">NewGuid</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
                withdrawal<span class="token punctuation">.</span>AccountId<span class="token punctuation">,</span>
                StatusEnum<span class="token punctuation">.</span>Failed<span class="token punctuation">,</span>
                withdrawal<span class="token punctuation">.</span>Amount<span class="token punctuation">,</span>
                <span class="token keyword">false</span><span class="token punctuation">,</span>
                ex<span class="token punctuation">.</span>Message<span class="token punctuation">,</span>
                TransactionTypeEnum<span class="token punctuation">.</span>Withdrawal<span class="token punctuation">,</span>
                DateTime<span class="token punctuation">.</span>Now<span class="token punctuation">)</span>
            <span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span><span class="token function">SaveChangesAsync</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token keyword">throw</span><span class="token punctuation">;</span>
        <span class="token punctuation">}</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>Now, let&rsquo;s analyze what is being done in both consumers.</p><p>The above classes <code class="inline-code">DepositConsumer</code> and <code class="inline-code">WithdrawalConsumer</code> use <strong>MassTransit</strong> to process events through the <code class="inline-code">IConsumer&lt;T&gt;</code> interface.</p><p>When a transaction event arrives, the <code class="inline-code">Consume()</code> method extracts the data from the received message and persists it to the database. In both classes, a new transaction status record (<code class="inline-code">TransactionStatus</code>) is created, making it possible to track the actions that occurred during processing.</p><p>After the transaction status is created, the deposit and withdrawal events are added to their database datasets (<code class="inline-code">_dbContext.Deposits</code> and <code class="inline-code">_dbContext.Withdrawals</code>). Finally, the changes are persisted in the database through the <code class="inline-code">SaveChangesAsync()</code> call.</p><h2 id="creating-the-controller-class">Creating the Controller Class</h2><p>In the controller class, we will add the endpoints for deposit and withdrawal that will call the events. So, create a new folder called &ldquo;Controllers&rdquo; and, inside it, add the following controller class:</p><ul><li>AccountController</li></ul><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Data<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Events<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Models<span class="token punctuation">;</span>
<span class="token keyword">using</span> MassTransit<span class="token punctuation">;</span>
<span class="token keyword">using</span> Microsoft<span class="token punctuation">.</span>AspNetCore<span class="token punctuation">.</span>Mvc<span class="token punctuation">;</span>

<span class="token keyword">namespace</span> BankStream<span class="token punctuation">.</span>Controllers<span class="token punctuation">;</span>

<span class="token punctuation">[</span>ApiController<span class="token punctuation">]</span>
<span class="token punctuation">[</span><span class="token function">Route</span><span class="token punctuation">(</span><span class="token string">"api/accounts"</span><span class="token punctuation">)</span><span class="token punctuation">]</span>
<span class="token keyword">public</span> <span class="token keyword">class</span> <span class="token class-name">AccountController</span> <span class="token punctuation">:</span> ControllerBase
<span class="token punctuation">{</span>
    <span class="token keyword">private</span> <span class="token keyword">readonly</span> IBus _bus<span class="token punctuation">;</span>
    <span class="token keyword">private</span> <span class="token keyword">readonly</span> EventDbContext _dbContext<span class="token punctuation">;</span>

    <span class="token keyword">public</span> <span class="token function">AccountController</span><span class="token punctuation">(</span>IBus bus<span class="token punctuation">,</span> EventDbContext dbContext<span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
        _bus <span class="token operator">=</span> bus<span class="token punctuation">;</span>
        _dbContext <span class="token operator">=</span> dbContext<span class="token punctuation">;</span>
    <span class="token punctuation">}</span>

    <span class="token punctuation">[</span><span class="token function">HttpPost</span><span class="token punctuation">(</span><span class="token string">"deposit"</span><span class="token punctuation">)</span><span class="token punctuation">]</span>
    <span class="token keyword">public</span> <span class="token keyword">async</span> Task<span class="token operator">&lt;</span>IActionResult<span class="token operator">&gt;</span> <span class="token function">Deposit</span><span class="token punctuation">(</span><span class="token punctuation">[</span>FromBody<span class="token punctuation">]</span> DepositEvent deposit<span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
        <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>TransactionStatus<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">TransactionStatus</span><span class="token punctuation">(</span>
            Guid<span class="token punctuation">.</span><span class="token function">NewGuid</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
            deposit<span class="token punctuation">.</span>AccountId<span class="token punctuation">,</span>
            StatusEnum<span class="token punctuation">.</span>Pending<span class="token punctuation">,</span>
            deposit<span class="token punctuation">.</span>Amount<span class="token punctuation">,</span>
            <span class="token keyword">false</span><span class="token punctuation">,</span>
            <span class="token keyword">string</span><span class="token punctuation">.</span>Empty<span class="token punctuation">,</span>
            TransactionTypeEnum<span class="token punctuation">.</span>Deposit<span class="token punctuation">,</span>
            DateTime<span class="token punctuation">.</span>Now<span class="token punctuation">)</span>
        <span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span><span class="token function">SaveChangesAsync</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

        <span class="token keyword">await</span> _bus<span class="token punctuation">.</span><span class="token function">Publish</span><span class="token punctuation">(</span>deposit<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">return</span> <span class="token function">Accepted</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>

    <span class="token punctuation">[</span><span class="token function">HttpPost</span><span class="token punctuation">(</span><span class="token string">"withdrawal"</span><span class="token punctuation">)</span><span class="token punctuation">]</span>
    <span class="token keyword">public</span> <span class="token keyword">async</span> Task<span class="token operator">&lt;</span>IActionResult<span class="token operator">&gt;</span> <span class="token function">Withdraw</span><span class="token punctuation">(</span><span class="token punctuation">[</span>FromBody<span class="token punctuation">]</span> WithdrawalEvent withdrawal<span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
        <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span>TransactionStatus<span class="token punctuation">.</span><span class="token function">AddAsync</span><span class="token punctuation">(</span><span class="token keyword">new</span> <span class="token class-name">TransactionStatus</span><span class="token punctuation">(</span>
             Guid<span class="token punctuation">.</span><span class="token function">NewGuid</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">,</span>
             withdrawal<span class="token punctuation">.</span>AccountId<span class="token punctuation">,</span>
             StatusEnum<span class="token punctuation">.</span>Pending<span class="token punctuation">,</span>
             withdrawal<span class="token punctuation">.</span>Amount<span class="token punctuation">,</span>
             <span class="token keyword">false</span><span class="token punctuation">,</span>
             <span class="token keyword">string</span><span class="token punctuation">.</span>Empty<span class="token punctuation">,</span>
             TransactionTypeEnum<span class="token punctuation">.</span>Withdrawal<span class="token punctuation">,</span>
             DateTime<span class="token punctuation">.</span>Now<span class="token punctuation">)</span>
         <span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">await</span> _dbContext<span class="token punctuation">.</span><span class="token function">SaveChangesAsync</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

        <span class="token keyword">await</span> _bus<span class="token punctuation">.</span><span class="token function">Publish</span><span class="token punctuation">(</span>withdrawal<span class="token punctuation">)</span><span class="token punctuation">;</span>
        <span class="token keyword">return</span> <span class="token function">Accepted</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>

   <span class="token punctuation">[</span><span class="token function">HttpGet</span><span class="token punctuation">(</span><span class="token string">"{accountId}/balance"</span><span class="token punctuation">)</span><span class="token punctuation">]</span>
    <span class="token keyword">public</span> IActionResult <span class="token function">GetBalance</span><span class="token punctuation">(</span><span class="token keyword">string</span> accountId<span class="token punctuation">)</span>
    <span class="token punctuation">{</span>
        <span class="token keyword">var</span> deposits <span class="token operator">=</span> _dbContext<span class="token punctuation">.</span>
            Deposits<span class="token punctuation">.</span>
            <span class="token function">Where</span><span class="token punctuation">(</span>d <span class="token operator">=</span><span class="token operator">&gt;</span> d<span class="token punctuation">.</span>AccountId <span class="token operator">==</span> accountId<span class="token punctuation">)</span>
            <span class="token punctuation">.</span><span class="token function">Sum</span><span class="token punctuation">(</span>d <span class="token operator">=</span><span class="token operator">&gt;</span> d<span class="token punctuation">.</span>Amount<span class="token punctuation">)</span><span class="token punctuation">;</span>

        <span class="token keyword">var</span> withdrawals <span class="token operator">=</span> _dbContext
            <span class="token punctuation">.</span>Withdrawals
            <span class="token punctuation">.</span><span class="token function">Where</span><span class="token punctuation">(</span>w <span class="token operator">=</span><span class="token operator">&gt;</span> w<span class="token punctuation">.</span>AccountId <span class="token operator">==</span> accountId<span class="token punctuation">)</span>
            <span class="token punctuation">.</span><span class="token function">Sum</span><span class="token punctuation">(</span>w <span class="token operator">=</span><span class="token operator">&gt;</span> w<span class="token punctuation">.</span>Amount<span class="token punctuation">)</span><span class="token punctuation">;</span>

        <span class="token keyword">return</span> <span class="token function">Ok</span><span class="token punctuation">(</span>deposits <span class="token operator">-</span> withdrawals<span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span>
<span class="token punctuation">}</span>
</code></pre><p>Here, the controller uses the <code class="inline-code">IBus</code> interface, which is the MassTransit message bus used to publish events.</p><p>When a client makes an HTTP POST request to the <code class="inline-code">/api/accounts/deposit</code> endpoint, the <code class="inline-code">Deposit()</code> method receives a <code class="inline-code">DepositEvent</code>, representing the request for a deposit. Before publishing the event to the bus, a transaction status of Pending is recorded in the database, indicating that the operation has been requested but has not yet been completed. After persisting this status, the event is published with <code class="inline-code">_bus.Publish(deposit)</code>, allowing asynchronous consumers such as <code class="inline-code">DepositConsumer</code> to process the transaction.</p><p>The same flow applies to the <code class="inline-code">Withdraw</code> method, which responds to the <code class="inline-code">/api/accounts/withdrawal</code> endpoint. It receives a <code class="inline-code">WithdrawalEvent</code>, records the transaction as Pending in the database, and publishes the event to the bus for later processing by the <code class="inline-code">WithdrawalConsumer</code>.</p><p>In addition to these two endpoints, there is a third one that is used to check the balance. To do this, instead of just fetching the value from the database as in traditional approaches, it uses the principle of Event Sourcing to reconstruct the current state through the transaction history. So, first, the totals of deposits and withdrawals are obtained, and the balance value is obtained by subtracting the total withdrawals from the total deposits.</p><h3 id="configuring-the-program-class">Configuring the Program Class</h3><p>In the Program class, we will configure the <code class="inline-code">EventDbContext</code> class to use the SQLite database. In addition, we will configure MassTransit and RabbitMQ to send and consume events.</p><p>So, open the Program class of your application and replace whatever is in it with the following code:</p><pre class=" language-csharp"><code class="prism  language-csharp"><span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Consumers<span class="token punctuation">;</span>
<span class="token keyword">using</span> BankStream<span class="token punctuation">.</span>Data<span class="token punctuation">;</span>
<span class="token keyword">using</span> MassTransit<span class="token punctuation">;</span>
<span class="token keyword">using</span> Microsoft<span class="token punctuation">.</span>EntityFrameworkCore<span class="token punctuation">;</span>

<span class="token keyword">var</span> builder <span class="token operator">=</span> WebApplication<span class="token punctuation">.</span><span class="token function">CreateBuilder</span><span class="token punctuation">(</span>args<span class="token punctuation">)</span><span class="token punctuation">;</span>

builder
    <span class="token punctuation">.</span>Services
    <span class="token punctuation">.</span><span class="token generic-method function">AddDbContext<span class="token punctuation">&lt;</span>EventDbContext<span class="token punctuation">&gt;</span></span><span class="token punctuation">(</span>options <span class="token operator">=</span><span class="token operator">&gt;</span> options<span class="token punctuation">.</span><span class="token function">UseSqlite</span><span class="token punctuation">(</span><span class="token string">"Data Source=events.db"</span><span class="token punctuation">)</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
builder
    <span class="token punctuation">.</span>Services
    <span class="token punctuation">.</span><span class="token function">AddMassTransit</span><span class="token punctuation">(</span>x <span class="token operator">=</span><span class="token operator">&gt;</span>
    <span class="token punctuation">{</span>
        x<span class="token punctuation">.</span><span class="token generic-method function">AddConsumer<span class="token punctuation">&lt;</span>DepositConsumer<span class="token punctuation">&gt;</span></span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        x<span class="token punctuation">.</span><span class="token generic-method function">AddConsumer<span class="token punctuation">&lt;</span>WithdrawalConsumer<span class="token punctuation">&gt;</span></span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
        x<span class="token punctuation">.</span><span class="token function">UsingRabbitMq</span><span class="token punctuation">(</span>
            <span class="token punctuation">(</span>context<span class="token punctuation">,</span> cfg<span class="token punctuation">)</span> <span class="token operator">=</span><span class="token operator">&gt;</span>
            <span class="token punctuation">{</span>
                cfg<span class="token punctuation">.</span><span class="token function">Host</span><span class="token punctuation">(</span><span class="token string">"rabbitmq://localhost"</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                cfg<span class="token punctuation">.</span><span class="token function">ReceiveEndpoint</span><span class="token punctuation">(</span>
                    <span class="token string">"event-queue"</span><span class="token punctuation">,</span>
                    e <span class="token operator">=</span><span class="token operator">&gt;</span>
                    <span class="token punctuation">{</span>
                        e<span class="token punctuation">.</span><span class="token function">SetQueueArgument</span><span class="token punctuation">(</span><span class="token string">"x-message-ttl"</span><span class="token punctuation">,</span> <span class="token number">500000</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
                        e<span class="token punctuation">.</span><span class="token generic-method function">ConfigureConsumer<span class="token punctuation">&lt;</span>DepositConsumer<span class="token punctuation">&gt;</span></span><span class="token punctuation">(</span>context<span class="token punctuation">)</span><span class="token punctuation">;</span>
                        e<span class="token punctuation">.</span><span class="token generic-method function">ConfigureConsumer<span class="token punctuation">&lt;</span>WithdrawalConsumer<span class="token punctuation">&gt;</span></span><span class="token punctuation">(</span>context<span class="token punctuation">)</span><span class="token punctuation">;</span>
                    <span class="token punctuation">}</span>
                <span class="token punctuation">)</span><span class="token punctuation">;</span>
            <span class="token punctuation">}</span>
        <span class="token punctuation">)</span><span class="token punctuation">;</span>
    <span class="token punctuation">}</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

builder<span class="token punctuation">.</span>Services<span class="token punctuation">.</span><span class="token function">AddControllers</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

<span class="token keyword">var</span> app <span class="token operator">=</span> builder<span class="token punctuation">.</span><span class="token function">Build</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

app<span class="token punctuation">.</span><span class="token function">MapControllers</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>

app<span class="token punctuation">.</span><span class="token function">Run</span><span class="token punctuation">(</span><span class="token punctuation">)</span><span class="token punctuation">;</span>
</code></pre><p>Let&rsquo;s analyze the code above.</p><p>After configuring SQLite, MassTransit is registered, where the two consumers are configured: <code class="inline-code">DepositConsumer</code> and <code class="inline-code">WithdrawalConsumer</code>. In addition, communication with RabbitMQ is established to transport the messages, for which the connection is established with a local host (<code class="inline-code">rabbitmq://localhost</code>).</p><p>Within the RabbitMQ configuration, a receiving endpoint called <code class="inline-code">event-queue</code> is defined. This queue is configured with the <code class="inline-code">SetQueueArgument()</code> method and a special argument, <code class="inline-code">x-message-ttl</code>, which determines the time to live of the messages within the queue. This lets us remove expired messages automatically after 500,000 milliseconds (500 seconds). This is done so that the messages can be checked in the RabbitMQ dashboard. Finally, the consumers are linked to the queue, so that any deposit or withdrawal event received by RabbitMQ will be forwarded to the appropriate consumer for processing.</p><h2 id="running-ef-migrations">Running EF Migrations</h2><p>To create the database and tables, run the EF Core commands:</p><ol><li>Command to install EF if you don&rsquo;t have it yet:</li></ol><pre class=" language-bash"><code class="prism  language-bash">dotnet tool <span class="token function">install</span> --global dotnet-ef
</code></pre><ol start="2"><li>Generate the Migration files:</li></ol><pre class=" language-bash"><code class="prism  language-bash">dotnet ef migrations add InitialCreate
</code></pre><ol start="3"><li>Run the persistence:</li></ol><pre class=" language-bash"><code class="prism  language-bash">dotnet ef database update
</code></pre><h2 id="setting-up-rabbitmq-in-docker">Setting up RabbitMQ in Docker</h2><p>To run the application, you need to have RabbitMQ running locally. To do this, you can use the command below to download the RabbitMQ image and run a Docker container of it:</p><pre class=" language-bash"><code class="prism  language-bash">docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
</code></pre><p>After running the command, you can check the RabbitQM container running on Docker Desktop:</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-04/rabbitmq-docker-container.png?sfvrsn=c1cade5_2" title="rabbitmq docker container" alt="RabbitMQ Docker container" /></p><h2 id="sending-and-consuming-events">Sending and Consuming Events</h2><p>Now run the application and request the endpoint below. This post uses Progress Telerik <a href="https://www.telerik.com/fiddler/fiddler-everywhere" target="_blank">Fiddler Everywhere</a> to make the requests.</p><p>POST - <code class="inline-code">http://localhost:PORT/api/accounts/deposit</code></p><pre class=" language-json"><code class="prism  language-json"><span class="token punctuation">{</span>
  <span class="token string">"id"</span><span class="token punctuation">:</span> <span class="token string">"a1a7c1f2-3d4e-42b9-9e77-5f1e7c8b6c2f"</span><span class="token punctuation">,</span>
  <span class="token string">"accountId"</span><span class="token punctuation">:</span> <span class="token string">"123456789"</span><span class="token punctuation">,</span>
  <span class="token string">"amount"</span><span class="token punctuation">:</span> <span class="token number">1300.90</span>
<span class="token punctuation">}</span>
</code></pre><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-04/fiddler-request.png?sfvrsn=15a097e1_2" title="fiddler request" alt="Fiddler request" /></p><p>Now, access the RabbitMQ homepage at the address indicated in Docker (you can use the default username and password <code class="inline-code">guest</code>). Then click on the &ldquo;Queues and Streams&rdquo; menu, then on &ldquo;Get messages&rdquo;, and finally &ldquo;Get message(s)&rdquo;.</p><p>So, you can check the data sent in the message, as well as other data such as operating system, etc., as shown in the images below:</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-04/rabbitmq-get-messages.png?sfvrsn=6f6bed28_2" title="rabbitmq get messages" alt="RabbitMQ get messages" /></p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-04/rabbitmq-message-details.png?sfvrsn=e7dfdace_2" title="rabbitmq message details" alt="RabbitMQ message details" /></p><p>Now let&rsquo;s run the withdrawal endpoint, so make a new request to the endpoint</p><p>POST - <code class="inline-code">http://localhost:PORT/api/accounts/withdraw</code></p><pre class=" language-json"><code class="prism  language-json"><span class="token punctuation">{</span>
<span class="token string">"id"</span><span class="token punctuation">:</span> <span class="token string">"fef6df04-df3f-46a7-99a7-6d9992dd3558"</span><span class="token punctuation">,</span>
<span class="token string">"accountId"</span><span class="token punctuation">:</span> <span class="token string">"123456789"</span><span class="token punctuation">,</span>
<span class="token string">"amount"</span><span class="token punctuation">:</span> <span class="token number">200.00</span>
<span class="token punctuation">}</span>
</code></pre><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-04/fiddler-request-withdrawal.png?sfvrsn=2d1a8727_2" title="fiddler request withdrawal" alt="Fiddler request withdrawal" /></p><p>Then, make another request to the endpoint below to retrieve the current balance:</p><p>GET - <code class="inline-code">http://localhost:PORT/api/accounts/123456789/balance</code></p><p>If everything goes well, you will get the result below as shown in the image.</p><p><img src="https://d585tldpucybw.cloudfront.net/sfimages/default-source/blogs/2025/2025-04/get-current-balance.png?sfvrsn=6e709326_2" title="get current balance" alt="Get current balance" /></p><p>Note that the resulting value was 1100.90. This value does not exist in the database but is the result of subtracting the deposit value of 1300.90 from the withdrawal value of 200.00.</p><h2 id="conclusion">Conclusion</h2><p>Whether or not to use Event Sourcing is a choice that should be made based on the needs of the project. If you need to keep a history of changes to a specific object, such as a bank account balance, for example, Event Sourcing can be a great choice. In addition, Event Sourcing has other advantages such as easy scalability, performance, decoupling, resilience and fault tolerance.</p><p>In this post, we understand how Event Sourcing differs from traditional approaches and how to implement a bank balance management system in practice, using the resources of RabbitMQ and MassTransit for sending and consuming messages.</p><p>So, I hope this post helps you understand and implement Event Sourcing using cutting-edge resources like ASP.NET Core, RabbitMQ, MassTransit and Docker.</p><aside><hr data-sf-ec-immutable="" /><div class="row"><div class="col-4 u-normal-full u-small-mb0"><h4 class="u-fs20 u-fw5 u-lh125 u-mb0">Make It Run Faster: Optimizing Your ASP.NET Core Application</h4></div><div class="col-8"><p class="u-fs16 u-mb0">If you aren&rsquo;t using all six of these <a target="_blank" href="https://www.telerik.com/blogs/make-run-faster-optimizing-aspnet-core-application">tips for speeding up your ASP.NET Core application</a>, your application isn&rsquo;t running as fast as it could. And implementing any of these will speed up your app enough that you users will notice.</p></div></div></aside><img src="https://feeds.telerik.com/link/23069/17063183.gif" height="1" width="1"/>
