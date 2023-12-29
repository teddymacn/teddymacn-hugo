---
author: Teddy
title: "FluentDataflow - Fluent Style TPL Dataflow"
date: "2017-10-22"
#description: ""
summary: ""
tags: ["dotnet", "dataflow"]
#ShowToc: true
#TocOpen: true
---

**Background**

In modern application development, there are so many scenarios of dataflow processing, such large volume data processing, image/video conversion, data migration, etc.

In .NET, thanks to Microsoft for providing the TPL Dataflow library which is a flexible and high performance library for this area.

TPL dataflow is really flexible, but when we want to build complex dataflows, we might have to create bunch of blocks, manually wire them, be careful to avoid data blocking, write a lot of tricks even for implementing the simplest branching and merging cases. If you are ever a dataflow developer, do you remember how many times you ever wait for your dataflow to complete but it never happens and you have to debug line by line to find out where the message is blocked and why.

And for long time, I heard of developers complaining about its lack of a fluent style API which is kind of standard configuration for almost all JAVA world libraies and also already provided by many popular .NET libraries. So when I also experienced the same fustratings, I decide to build one, which is what I want to introduce here - [FluentDataflow](https://github.com/teddymacn/FluentDataflow).

**Introduction**

[FluentDataflow](https://github.com/teddymacn/FluentDataflow), in short, is a fluent style wrapper and extension of TPL Dataflow.

The only entry-point of this library is the IDataflowFactory interface. Through the fluent style API provided by a factory instance, the outputs of this factory are still instances of standard IDataflowBlock, such as ITargetBlock, ISourceBlock, IPropogatorBlock, etc, instances, each of which usually wraps a simple or even a complex custom dataflow.

I heard what you are shouting: "Show me the code!" So, let's go!

**Examples**

- Simple aggregation example:

``` csharp
var splitter = new TransformBlock<string, KeyValuePair<string, int>>(input =>
{
    string[] splitted = input.Split('=');
    return new KeyValuePair<string, int>(splitted[0], int.Parse(splitted[1]));
});

var dict = new Dictionary<string, int>();

var aggregater = new ActionBlock<KeyValuePair<string, int>>(pair =>
{
    int oldValue;
    dict[pair.Key] = dict.TryGetValue(pair.Key, out oldValue) ? oldValue + pair.Value : pair.Value;
});

// the created the dataflow instance is also
// a strong-typed standard dataflow block
var dataflow = new DataflowFactory().FromPropagator(splitter)
    .LinkToTarget(aggregater)
    .Create();

dataflow.Post("a=1");
dataflow.Post("b=2");
dataflow.Post("a=5");
dataflow.Complete();

Task.WaitAll(dataflow.Completion);

System.Console.WriteLine("sum(a) = {0}", dict["a"]); //prints sum(a) = 6
```

- A dataflow as part of another dataflow example

``` csharp
var factory = new DataflowFactory();

var aggregater = ... // Build an aggregator dataflow
var splitter = new TransformManyBlock<string, string>(line => line.Split(' '));

var dataflow = factory.FromPropagator<string, string>(splitter)
    .LinkToTarget(aggregator)
    .Create();

dataflow.Post("a=1 b=2 a=5");
dataflow.Post("c=6 b=8");
dataflow.Complete();

Task.WaitAll(dataflow.Completion);

System.Console.WriteLine("sum(b) = {0}", result["b"]); //prints sum(b) = 10
```

- Broadcast example

``` csharp
var printer1 = new ActionBlock<string>(s => System.Console.WriteLine("Printer1: {0}", s));
var printer2 = new ActionBlock<string>(s => System.Console.WriteLine("Printer2: {0}", s));
var printer3 = new ActionBlock<string>(s => System.Console.WriteLine("Printer3: {0}", s));

var dataflow = new DataflowFactory().FromBroadcast<string>()
    .LinkTo(printer1)
    .LinkTo(printer2)
    .LinkTo(printer3)
    .Create();

dataflow.Post("first message");
dataflow.Post("second message");
dataflow.Post("third message");
dataflow.Complete();

Task.WaitAll(dataflow.Completion);
```

- Multiple sources example

``` csharp
// by default, with native TPL, if multiple sources link to the same target,
// if set PropagateCompletion=true,
// as long as one of the source complets, the target complets.
// the target will miss some of the messages from the other sources.

// BUT, with our dataflow factory here, when set PropagateCompletion=true,
// dataflow.Complete() internally waits for all the sources to complete,
// so the target is guaranteed to receive all the messages from all the sources

var source1 = new BufferBlock<string>();
var source2 = new BufferBlock<string>();
var source3 = new BufferBlock<string>();
var printer = new ActionBlock<string>(s => System.Console.WriteLine(s));

var dataflow = new DataflowFactory().FromMultipleSources(source1, source2, source3)
    .LinkToTarget(printer)
    .Create();

for (var i = 0; i < 3; ++i)
{
    var s = i.ToString();
    source1.Post(s);
    source2.Post(s);
    source3.Post(s);
}

dataflow.Complete();

Task.WaitAll(dataflow.Completion);
```

- Linking with filter example

``` csharp
// the filter only accepts even numbers,
// so odd numbers goes to declined printer
var filter = new Predicate<int>(i =>
{
    return i % 2 == 0;
});
var printer = new ActionBlock<int>(s => System.Console.WriteLine("printer: " + s.ToString()));
var declinedPrinter = new ActionBlock<int>(s => System.Console.WriteLine("declined: " + s.ToString()));
var inputBlock = new BufferBlock<int>();

var dataflow = new DataflowFactory().FromPropagator(inputBlock)
    .LinkToTarget(printer
        , filter
        // when linking with filter, you have to specify a declined block
        // otherwise, because there will be messages declined still in the queue,
        // the current block will not be able to COMPLETE (waits on its Completion will never return)
        , declinedPrinter)
    .Create();

for (int i = 0; i < 10; ++i)
{
    dataflow.Post(i);
}

dataflow.Complete();

Task.WaitAll(dataflow.Completion);
```
- Join example

``` csharp
var source1 = new BufferBlock<string>();
var source2 = new BufferBlock<string>();
var printer = new ActionBlock<Tuple<string, string>>(s => System.Console.WriteLine("printer: {0},{1}", s.Item1, s.Item2));

var dataflow = new DataflowFactory().Join(source1, source2)
    .LinkToTarget(printer)
    .Create();

for (var i = 0; i < 3; ++i)
{
    var s = i.ToString();
    source1.Post(s);
    source2.Post(s);
}

dataflow.Complete();

Task.WaitAll(dataflow.Completion);
```

- Batch example

``` csharp
var source = new BufferBlock<string>();
var printer = new ActionBlock<IEnumerable<string>>(s => System.Console.WriteLine("printer: " +string.Join("|", s)));

var dataflow = new DataflowFactory().FromSource(source)
    .Batch(2)
    .LinkToTarget(printer)
    .Create();

for (var i = 0; i < 6; ++i)
{
    var s = i.ToString();
    source.Post(s);
}

dataflow.Complete();

Task.WaitAll(dataflow.Completion);
```
- If-else branching and merging example

``` csharp
var factory = new DataflowFactory();
// the ifFilter only accepts even numbers,
// so odd numbers goes to elseBlock
var ifFilter = new Predicate<int>(i =>
{
    return i % 2 == 0;
});
var printer = new ActionBlock<int>(s => System.Console.WriteLine("printer: " + s.ToString()));
var inputBlock = new BufferBlock<int>();
// if meet ifFilter, convert to: i -> i * 10
var ifBlock = new TransformBlock<int, int>(i => i * 10);
// else, convert to: i -> i * 100
var elseBlock = new TransformBlock<int, int>(i => i * 100);

var branchingDataflow = factory.FromPropagator(inputBlock)
    .LinkToPropagator(ifBlock, ifFilter, elseBlock)
    .Create();

var mergeingDataflow = factory.FromMultipleSources(ifBlock, elseBlock)
    .LinkToTarget(printer)
    .Create();

//encapsulate branchingDataflow and mergeingDataflow
var dataflow = factory.EncapsulateTargetDataflow(branchingDataflow, mergeingDataflow);

for (int i = 0; i < 10; ++i)
{
    dataflow.Post(i);
}

dataflow.Complete();

Task.WaitAll(dataflow.Completion);
```
