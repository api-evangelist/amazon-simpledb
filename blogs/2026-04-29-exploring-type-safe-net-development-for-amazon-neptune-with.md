---
title: "Exploring type-safe .NET development for Amazon Neptune with Gremlinq"
url: "https://aws.amazon.com/blogs/database/exploring-type-safe-net-development-for-amazon-neptune-with-gremlinq/"
date: "Wed, 29 Apr 2026 15:25:35 +0000"
author: "Stephen Mallette"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
<p>When you build graph database applications on <a href="https://aws.amazon.com/neptune/" rel="noopener noreferrer" target="_blank">Amazon Neptune</a> using .NET, one of the early architectural decisions that you will make is how to construct and execute <a href="https://tinkerpop.apache.org/" rel="noopener noreferrer" target="_blank">Apache TinkerPop</a> based <a href="https://docs.aws.amazon.com/neptune/latest/userguide/access-graph-gremlin.html" rel="noopener noreferrer" target="_blank">Gremlin</a> queries. You can work directly with Gremlin’s <a href="https://tinkerpop.apache.org/docs/current/reference/#gremlin-dotnet" rel="noopener noreferrer" target="_blank">traversal API</a>, writing queries that use string-based property names and handling dynamic results. Alternatively, you can use an abstraction layer that provides wider support for compile-time type checking and strongly typed result objects. In making these decisions, you will want to understand your options and their trade-offs.</p> 
<p>Type-safety in graph database development with .NET means catching certain categories of errors, like misspelled property names, at compile time rather than discovering them during testing or in production. It also means working with familiar C# patterns like Language Integrated Query (LINQ) expressions and lambda syntax to build queries. The main trade-off is an additional abstraction layer between your code and the underlying Gremlin traversals.</p> 
<p><a href="https://docs.gremlinq.net/" rel="noopener noreferrer" target="_blank">ExRam.Gremlinq</a> is an open source Object-Graph-Mapper (OGM) that provides a strongly typed development experience for TinkerPop-enabled graph databases. Since its creation in 2017, the maintainers actively develop it to translate C# queries into Gremlin traversals while preserving type information throughout the query pipeline. If you value compile-time verification and IDE-assisted development, or those working on large code bases where refactoring safety matters, Gremlinq offers an alternative worth evaluating.</p> 
<p>In this post, we walk through how Gremlinq works, demonstrate its capabilities, show you how to set up a Neptune project with the provided templates, and help you understand where this approach might fit in your development context.</p> 
<h2>Working with Gremlin in .NET</h2> 
<p>Let’s first look at some characteristics of working with Gremlin.NET and how they relate to typical C# development patterns. When using Gremlin.NET directly, you generally construct queries as strings or through a fluent API that still requires string-based labels and property names:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Gremlin.NET approach - string-based query
var query = "g.V().hasLabel('Airport').has('Code','SEA').out('Route').hasLabel('Airport')";
var result = await gremlinClient.SubmitAsync&lt;dynamic&gt;(query);
// Or using the fluent API - still requires string labels and property names
var traversal = g.V()
    .HasLabel("Airport")
    .Has("code", "SEA")
    .Out("Route")
    .HasLabel("Airport");</code></pre> 
</div> 
<p>When you work directly with Gremlin.NET, you will notice a few characteristics that shape the development experience. If you misenter “Airport” as “aiport”, you will catch that at runtime rather than during compilation. The queries return dynamic objects, which means that you will handle manual casting and work without the compile-time guidance that strongly typed projections provide. Property names like “code” appear as strings throughout your code, so typos and refactoring require careful attention. Because the API doesn’t encode the current traversal shape in its type system, it’s possible to compose traversals that are syntactically valid after calling V() but might not make sense at that point in your query.</p> 
<p>Results typically come back as dictionaries or dynamic objects that you then map into your domain classes:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var results = await gremlinClient.SubmitAsync&lt;dynamic&gt;(query);
foreach (var result in results)
{
    // Manual property extraction, which can be repetitive in larger applications
    var code = result["code"];
    var city = result["city"];
    // ... build your Airport object manually
}</code></pre> 
</div> 
<p>This direct approach to query construction works well for many applications and, as you will see later, can sometimes be necessary. The next section introduces how Gremlinq works as an alternative.</p> 
<h2>Gremlinq: A Typed OGM for Gremlin</h2> 
<p>Gremlinq takes a different approach where it presents a type system that carries element information throughout the query, C# expression support for building traversals, and automatic, type‑safe deserialization of results. To understand how these capabilities work in practice, we examine the core features that enable this alternative development experience.</p> 
<h3>Type-safe domain modeling</h3> 
<p>With Gremlinq, you define your graph elements as plain C# classes:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">public sealed class Airport : Vertex
{
    public string? Code { get; set; }
    public string? ICAO { get; set; }
    public string? City { get; set; }
    public string? Region { get; set; }
    public string? Country { get; set; }
    public string? Description { get; set; }
    public int Runways { get; set; }
    public int Elevation { get; set; }
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public int LongestRunway { get; set; }
}
public sealed class Route : Edge
{
    public long Distance { get; set; }
}</code></pre> 
</div> 
<p>These are regular POCOs (Plain Old C# Objects) with no special attributes or base classes required beyond inheriting from a marker base type. The property names map directly to graph properties, and the class name becomes the vertex or edge label.</p> 
<h3>Fluent, type-aware queries</h3> 
<p>The previously mentioned Gremlin.NET query for routes out of Seattle would look like this in Gremlinq:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var destinationsFromSeattle = await _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code == "SEA")
    .Out&lt;Route&gt;()
    .OfType&lt;Airport&gt;()
    .ToArrayAsync();</code></pre> 
</div> 
<p>The Gremlinq version expresses the same traversal using strongly typed C# constructs with compile‑time type information throughout the query:</p> 
<ul> 
 <li><code>V&lt;Airport&gt;()</code> returns an <code>IVertexGremlinQuery&lt;Airport&gt;</code></li> 
 <li>The <code>.Where()</code> method accepts a lambda expression with IntelliSense for <code>Airport</code> properties</li> 
 <li><code>.Out&lt;Route&gt;()</code> traverses edges of type <code>Route</code></li> 
 <li><code>.OfType&lt;Airport&gt;()</code> narrows the result to airports</li> 
 <li>The result is of type <code>Airport[]</code>, an array of <code>Airport</code>. No casting needed</li> 
</ul> 
<h3>C# expression recognition</h3> 
<p>One of Gremlinq’s most important features is its ability to recognize standard C# expressions and translate them to the appropriate Gremlin steps. You write familiar C# code, and Gremlinq handles the translation:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// String operations
.Where(airport =&gt; airport.Code!.StartsWith("S"))    // Translates to TextP.startingWith()
// Numeric comparisons
.Where(route =&gt; route.Distance &lt;= 1500)          // Translates to P.lte(1500)
// Equality checks
.Where(airport =&gt; airport.Code == "SEA")             // Translates to has('code', 'SEA')</code></pre> 
</div> 
<p>This allows Gremlinq to <a href="https://docs.gremlinq.net/reference/where_expressions/" rel="noopener noreferrer" target="_blank">translate many common C# expressions</a> into the corresponding Gremlin predicates, like <code>P.eq()</code>, <code>P.gte()</code>, or <code>TextP.containing()</code>, so you can use familiar C# syntax while Gremlinq generates the underlying Gremlin steps.</p> 
<h3>IntelliSense-guided development</h3> 
<p>Because the query type changes based on the traversal steps that you’ve applied, your IDE can offer context-appropriate suggestions. After calling <code>.V&lt;Airport&gt;()</code>, IntelliSense shows you vertex-specific operations. After calling <code>.OutE&lt;Route&gt;()</code>, you see edge-specific operations. This guided experience can help you to discover valid traversal operations and to avoid certain categories of invalid query construction.</p> 
<h2>Prerequisites</h2> 
<p>In the following sections, we set up a basic Gremlinq project and show some example usage with Neptune. You need <a href="https://dotnet.microsoft.com/en-us/download/dotnet/10.0" rel="noopener noreferrer" target="_blank">Microsoft .NET 10</a> or higher installed to run the project template and, if desired, a <a href="https://docs.aws.amazon.com/neptune/latest/userguide/neptune-setup.html" rel="noopener noreferrer" target="_blank">Neptune instance</a> with the <a href="https://krlawrence.github.io/graph/#air" rel="noopener noreferrer" target="_blank">Air Routes dataset</a> loaded to run the queries. One of the most direct ways to load that dataset is through the <a href="https://github.com/aws/graph-notebook/blob/main/src/graph_notebook/notebooks/01-Neptune-Database/02-Visualization/Air-Routes-Gremlin.ipynb" rel="noopener noreferrer" target="_blank">Graph Notebook</a>.</p> 
<h2>Set up a Neptune project</h2> 
<p>Gremlinq includes .NET templates that scaffold a complete project with sample queries. As a next step, we walk through creating a console application targeting Amazon Neptune.</p> 
<h3>Installing the templates</h3> 
<p>First, install the Gremlinq project templates:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">dotnet new install ExRam.Gremlinq.Templates</code></pre> 
</div> 
<h3>Creating a Neptune project</h3> 
<p>Create a new console project configured for Amazon Neptune:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">dotnet new gremlinq-console --provider Neptune -n MyNeptuneApp
cd MyNeptuneApp</code></pre> 
</div> 
<p>This generates a fully functional console application with sample queries using an airline routes domain model.</p> 
<h3>Configuring the Neptune connection</h3> 
<p>Open <code>Program.cs</code> and configure the connection to your Neptune endpoint:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">using ExRam.Gremlinq.Core;
using ExRam.Gremlinq.Providers.Neptune;
using ExRam.Gremlinq.Support.NewtonsoftJson;
public class Program
{
    private readonly IGremlinQuerySource _g;
    public Program()
    {
        var endpoint = new Uri("wss://your-neptune-endpoint:port/gremlin");
        _g = GremlinQuerySource.g
            .UseNeptune&lt;Vertex, Edge&gt;(configurator =&gt; configurator
                .At(endpoint)
                .UseIAMAuthentication(iam =&gt; iam
                    .UseSigV4()
                    .WithUri(endpoint)
                    .WithRegion("us-east-1")
                    .WithCredentialsFromDefaultAWSCredentialsIdentityResolver())
                .UseNewtonsoftJson());    
    }
	// ... 
}</code></pre> 
</div> 
<p>The configuration is fluent and type-safe without magic strings for configuration keys. We use the <code>WithCredentialsFromDefaultAWSCredentialsIdentityResolver</code>, which walks the standard <a href="https://docs.aws.amazon.com/sdk-for-net/v4/developer-guide/creds-assign.html" rel="noopener noreferrer" target="_blank">AWS default credential chain</a> to fetch AWS credentials for authentication. The <code>.UseNeptune&lt;Vertex, Edge&gt;()</code> call tells Gremlinq about your base vertex and edge types, enabling the type system’s validation features.</p> 
<h3>Exploring the sample queries</h3> 
<p>The generated project includes a variety of sample queries that demonstrate Gremlinq’s capabilities. In the following sections, we explore them in depth and will start by adding some data to Neptune. The template includes a method to populate your database with sample airport and route data:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">await _g
    .CreateAirRoutesSmall();</code></pre> 
</div> 
<p>This method is idempotent. It checks for existing data before inserting, so you can call it safely on every run during development.</p> 
<h4>Retrieving all airports:</h4> 
<p>To start, we recommend getting a list of all airports:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var airports = await _g
    .V&lt;Airport&gt;()
    .ToArrayAsync();</code></pre> 
</div> 
<p>The semantics of this is straightforward: Start at all vertices of type <code>Airport</code> and materialize them as an array. After awaiting the query, the result is an object of type <code>Airport[]</code> with all members populated.</p> 
<h4>Filtering with C# expressions:</h4> 
<p>Because we don’t want to retrieve all the airports all the time, let’s apply a filter on the query that will only let those airports pass whose code starts with the letter ‘S’:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var airportCodesStartingWithLetterS = await _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code!.StartsWith("S"))
    .ToArrayAsync();</code></pre> 
</div> 
<p>There is something natural in how this feels in that you are using the familiar <code>string.StartsWith()</code> call and Gremlinq will translate it to the appropriate Gremlin text predicate under the hood.</p> 
<h4>Finding destinations from Seattle:</h4> 
<p>Graph databases excel at relationship traversals. Here’s how Gremlinq handles it:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var destinationsFromSeattle = await _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code == "SEA")
    .Out&lt;Route&gt;()
    .OfType&lt;Airport&gt;()
    .ToArrayAsync();</code></pre> 
</div> 
<p>The following listing breaks this query down step by step:</p> 
<ol> 
 <li><code>.V&lt;Airport&gt;()</code> – Start with airport vertices</li> 
 <li><code>.Where(airport =&gt; airport.Code == "SEA")</code> – Filter on Seattle</li> 
 <li><code>.Out&lt;Route&gt;()</code> – Traverse outgoing <code>Route</code> edges</li> 
 <li><code>.OfType&lt;Airport&gt;()</code> – Checks results are of type <code>Airport</code> (they will be in any case given the schema, but the explicit <code>OfType</code> confirms it and reestablishes type information)</li> 
 <li><code>.ToArrayAsync()</code> – Execute and return <code>Airport[]</code></li> 
</ol> 
<h4>Finding incoming routes:</h4> 
<p>Swap <code>.Out&lt;Route&gt;()</code> for <code>.In&lt;Route&gt;()</code> to reverse the direction we’re walking:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var routesIntoSEA = await _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code == "SEA")
    .In&lt;Route&gt;()
    .OfType&lt;Airport&gt;()
    .ToArrayAsync();</code></pre> 
</div> 
<p>What we get is a list of all the airports that can reach SEA by a direct route.</p> 
<h4>Working with edge properties</h4> 
<p>When you need to filter or access edge properties, use <code>.OutE()</code> and <code>.InE()</code> to traverse to the edges themselves:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var within1500Miles = await _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code == "SEA")
    .OutE&lt;Route&gt;()
    .Where(route =&gt; route.Distance &lt;= 1500)
    .InV&lt;Airport&gt;()
    .ToArrayAsync();</code></pre> 
</div> 
<p>This query:</p> 
<ol> 
 <li>Starts at SEA</li> 
 <li>Traverses to the outgoing <code>Route</code> edges (not directly to vertices)</li> 
 <li>Filters routes where <code>Distance &lt;= 1500</code></li> 
 <li>Traverses from those edges to their target airports</li> 
</ol> 
<p>The type system tracks everything: after <code>.OutE&lt;Route&gt;()</code>, you have access to <code>Route</code> properties like <code>Distance</code>. After <code>.InV&lt;Airport&gt;()</code>, you’re back to working with airports.</p> 
<h4>Working with subqueries</h4> 
<p>Graph traversals become more interesting when you need to explore multiple path lengths. For example, finding airports reachable by one or two connecting flights with the help of the <code>Union</code> operator:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var withinOneOrTwoFlights = await _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code == "SEA")
    .Union(
        __ =&gt; __
            .Out&lt;Route&gt;(),
        __ =&gt; __
            .Out&lt;Route&gt;()
            .Out&lt;Route&gt;())
    .OfType&lt;Airport&gt;()
    .Dedup()
    .ToArrayAsync();</code></pre> 
</div> 
<p>The <code>Union</code> operator runs multiple traversals in parallel and combines their results. Note the syntax for sub-queries: Each sub-query is denoted by a delegate, where in the case of the <code>Union</code>-operator (and most other operators dealing with sub-queries) the parameter of the delegate (<code>__</code>) is of the same query type as the query up to the call of <code>Union</code> (<code>IVertexGremlinQuery&lt;Airport&gt;</code> in this case). You can use this to deal with complete type information, even in a sub-query.</p> 
<p>The <code>Dedup()</code> step removes duplicates (an airport reachable by one flight might also be reachable by two).</p> 
<h4>Filtering with sub-queries</h4> 
<p>Sometimes you must filter based on characteristics discovered by further traversal:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var destinationsWithRoutesToAtlanta = await _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code == "SEA")
    .Out&lt;Route&gt;()
    .OfType&lt;Airport&gt;()
    .Where(__ =&gt; __
        .Out&lt;Route&gt;()
        .OfType&lt;Airport&gt;()
        .Where(airport =&gt; airport.Code == "ATL"))
    .ToArrayAsync();</code></pre> 
</div> 
<p>This finds airports reachable from Seattle that <em>also</em> have a direct route to Atlanta. The inner <code>.Where()</code> accepts a sub-query, if and only if that query yields at least one result element, the outer element passes the filter.</p> 
<h4>Ordering results</h4> 
<p>This is where we introduce the first notion of some kind of Domain Specific Language (DSL) with Gremlinq itself: When ordering results, you can use the Gremlin language to call <code>order</code>-operators and <code>by</code>-modulators on the same level. Gremlinq, however, allows for a fluent ordering API:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var orderedByCode = await _g
    .V&lt;Airport&gt;()
    .Order(o =&gt; o
        .By(airport =&gt; airport.Code))
    .ToArrayAsync();</code></pre> 
</div> 
<p>By entering Gremlinq’s Order DSL, we keep the <code>By</code>-modulators solely within this DSL. They won’t be available to you outside of this kind of DSL.</p> 
<p>The second example orders airports by their longest route distance—a computed value that requires traversing to edges. Gremlinq handles this with nested order builders:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var orderedByLongestRoute = await _g
    .V&lt;Airport&gt;()
    .Order(o =&gt; o
        .ByDescending(__ =&gt; __
            .OutE&lt;Route&gt;()
            .Order(o =&gt; o
                .ByDescending(route =&gt; route.Distance))
            .Values(route =&gt; route.Distance)))
    .ToArrayAsync();</code></pre> 
</div> 
<p>This also works for multiple ordering criteria:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var orderedByLongestRouteThenCode = await _g
    .V&lt;Airport&gt;()
    .Order(o =&gt; o
        .ByDescending(__ =&gt; __
            .OutE&lt;Route&gt;()
            .Order(o =&gt; o
                .ByDescending(route =&gt; route.Distance))
            .Values(route =&gt; route.Distance))
        .By(airport =&gt; airport.Code))
    .ToArrayAsync();</code></pre> 
</div> 
<h4>Limiting results</h4> 
<p>If result sets are expected to be large, we recommend that you apply some limitation on the size of the materialized results. Standard pagination operations are fully supported:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var fiveAirportsOrderedByLongestRoute = await _g
    .V&lt;Airport&gt;()
    .Order(o =&gt; o
        .ByDescending(__ =&gt; __
            .OutE&lt;Route&gt;()
            .Order(o =&gt; o
                .ByDescending(route =&gt; route.Distance))
            .Values(route =&gt; route.Distance)))
    .Limit(5)
    .ToArrayAsync();</code></pre> 
</div> 
<p>Other pagination patterns can also be achieved with <code>.Range(...)</code>, <code>.Skip(...)</code>, and <code>.Tail(...)</code> having synonymous features to the Gremlin steps of the same names.</p> 
<h4>Step Labels: Capturing and referencing traversal state</h4> 
<p>In Gremlin, step labels are represented as strings that the compiler doesn’t validate, so developers typically track their meaning by convention. When using Gremlinq, there’s a generic <code>StepLabel&lt;..&gt;</code> type that encodes the element type and query type, making it possible to work with step labels in a type‑safe way.</p> 
<p>The following query finds airports reachable from Seattle with exactly two flights but excludes Seattle itself from the results. The <code>.As(...)</code> method captures the current element (Seattle) in a <code>StepLabel&lt;IVertexGremlinQuery&lt;Airport&gt;, Airport&gt;</code>, which preserves not only element-type information but also query-type information. The <code>As(...)</code>-method handles the creation of such a <code>StepLabel</code> instance and passes it to the continuation, where it can be referenced later in the <code>.Where()</code> clause:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var withinTwoFlightsWithNoReturn = await _g
    .V&lt;Airport&gt;()
    .Where(departure =&gt; departure.Code == "SEA")
    .As((__, sea) =&gt; __
        .Out&lt;Route&gt;()
        .Out&lt;Route&gt;()
        .OfType&lt;Airport&gt;()
        .Where(destination =&gt; destination != sea.Value))
    .Dedup()
    .ToArrayAsync();</code></pre> 
</div> 
<h4>Projections</h4> 
<p>You can use Projections to shape query results into custom structures that go beyond dictionaries. This is useful when you need specific properties or computed values rather than entire vertices. There is a dedicated sub-DSL in Gremlinq for this:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var projectedTuple = await _g
    .V&lt;Airport&gt;()
    .Project(p =&gt; p
        .ToTuple()
        .By(x =&gt; x.Description!)
        .By(x =&gt; x.Code!)
        .By(__ =&gt; __.Out&lt;Route&gt;().Count()))
    .FirstAsync();</code></pre> 
</div> 
<p>This returns tuples containing each airport’s code, city, and the count of outgoing routes. You can also project into anonymous types or custom Data Transfer Objects (DTOs) for more complex scenarios.</p> 
<h4>Aggregating with Fold and Unfold</h4> 
<p>Aggregation of elements yielded throughout a graph-walk is done using the <code>fold</code>-operator in Gremlin. Gremlinq’s <code>.Fold()</code>method mirrors this while keeping the original element and query-type information intact:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var fold = await _g
    .V&lt;Airport&gt;()
    .Map(__ =&gt; __
        .Out&lt;Route&gt;()
        .OfType&lt;Airport&gt;()
        .Fold())
    .ToArrayAsync();</code></pre> 
</div> 
<p>The return type of Fold includes both the array element type and the underlying query type (for example, <code>IArrayGremlinQuery&lt;Airport[], Airport, IVertexGremlinQuery&lt;Airport&gt;&gt;</code> in this sample), so that subsequent unfolding can still work with rich type information.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var foldFilterUnfold = await _g
    .V&lt;Airport&gt;()
    .Map(__ =&gt; __
        .Out&lt;Route&gt;()
        .OfType&lt;Airport&gt;()
        .Fold()
        .Unfold()
        .Values(x =&gt; x.Code!))
    .ToArrayAsync();</code></pre> 
</div> 
<h4>Grouping</h4> 
<p>You can use grouping to organize results by a key, paired with the values that apply for that key. The following query groups all airports by the number of routes, and returns a dictionary where keys are integers and values are arrays of airports.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var groupByNumberOfRoutes = await _g
    .V&lt;Airport&gt;()
    .Group(g =&gt; g
        .ByKey(__ =&gt; __
            .OutE&lt;Route&gt;()
            .Count()))
    .ToArrayAsync();</code></pre> 
</div> 
<p>Implicitly, unless specified differently, the values are of the same type as the source type (in our case, <code>Airport</code>). We can specify this to be the airport code rather than the airport itself:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var groupCodesByNumberOfRoutes = await _g
    .V&lt;Airport&gt;()
    .Group(g =&gt; g
        .ByKey(__ =&gt; __
            .OutE&lt;Route&gt;()
            .Count())
        .ByValue(__ =&gt; __
            .Values(x =&gt; x.Code!)))
    .ToArrayAsync();</code></pre> 
</div> 
<h4>Loops with Repeat</h4> 
<p>For recursive traversals like finding paths or exploring hierarchies, Gremlin allows specifying repeated loops along emit-directives and exit-conditions. Gremlinq users can enter a dedicated DSL for looping by calling the <code>Loop</code>-method and going from there:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var threeFlights = await _g
    .V&lt;Airport&gt;()
    .Map(__ =&gt; __
        .Loop(ls =&gt; ls
            .Repeat(__ =&gt; __
                .Out&lt;Route&gt;()
                .OfType&lt;Airport&gt;())
            .Times(3))
        .Dedup()
        .Values(x =&gt; x.Code!)
        .Fold())
    .ToArrayAsync();</code></pre> 
</div> 
<p>You can also use <code>.Until()</code> to repeat until a condition is met, in this case, we walk <code>Route</code>-edges until we reach Atlanta (we restrict our result set by <code>Limit(...)</code> because the result set would become too big quickly):</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var repeatEmitUntilAtlanta = await _g
    .V&lt;Airport&gt;()
    .Where(x =&gt; x.Code == "SEA")
    .Loop(ls =&gt; ls
        .Repeat(__ =&gt; __
            .Out&lt;Route&gt;()
            .OfType&lt;Airport&gt;())
        .Emit()
        .Until(__ =&gt; __
            .Where(a =&gt; a.Code == "ATL")))
    .Dedup()
    .Limit(10)
    .Values(x =&gt; x.Code!)
    .ToArrayAsync();</code></pre> 
</div> 
<h4>Tree queries</h4> 
<p>You can use the Tree-operator in Gremlinq to construct tree-shaped result sets representing traversal paths in a graph, similar to Gremlin’s tree step. While the standard <code>Tree()</code> method returns results with limited compile-time type guarantees, Gremlinq enhances static type-safety through its generic <code>Tree()</code> overload. By specifying the element type in <code>Tree()</code>, you verify that the tree contains values of the expected type throughout the traversal.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var untypedTree = await _g
    .V&lt;Airport&gt;()
    .Where(a =&gt; a.Code == "SEA")
    .Out&lt;Route&gt;()
    .Out&lt;Route&gt;()
    .Tree()
    .FirstAsync();
var typedTree = await _g
    .V&lt;Airport&gt;()
    .Where(a =&gt; a.Code == "SEA")
    .Out&lt;Route&gt;()
    .Out&lt;Route&gt;()
    .Tree&lt;Airport&gt;()
    .FirstAsync();</code></pre> 
</div> 
<p>Additionally, type-safety can be further improved by using the <code>Of</code> and <code>By</code> modulators with <code>Tree</code>, allowing explicit selection of result projections and key types. These features can help catch certain type mismatches at compile time and make traversal expressions more predictable in strongly typed C#.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">var typedTreeWithOf = await _g
    .V&lt;Airport&gt;()
    .OutE&lt;Route&gt;()
    .InV&lt;Airport&gt;()
    .Tree(_ =&gt; _
        .Of&lt;Airport&gt;()
        .Of&lt;Route&gt;()
        .Of&lt;Airport&gt;())
    .FirstAsync();
var typedTreeWithOfAndBy = await _g
    .V&lt;Airport&gt;()
    .Out&lt;Route&gt;()
    .OfType&lt;Airport&gt;()
    .Tree(_ =&gt; _
        .Of&lt;Airport&gt;().By(x =&gt; x.Code!)
        .Of&lt;Airport&gt;().By(x =&gt; x.Description!))
    .FirstAsync();</code></pre> 
</div> 
<h2>ASP.NET integration</h2> 
<p>You can use Gremlinq in web applications with its built-in support for ASP.NET Core with dependency injection (DI). A separate template is available:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">dotnet new gremlinq-aspnet --provider Neptune -n MyNeptuneWebApp</code></pre> 
</div> 
<p>This scaffolds a Web API project with <code>IGremlinQuerySource</code> registered in the DI container. Configuration is loaded from <code>appsettings.json</code>, and controllers can inject the query source:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">[ApiController]
[Route("/airports")]
public class AirportsController : ControllerBase
{
    private readonly IGremlinQuerySource _g;
    public AirportsController(IGremlinQuerySource g)
    {
        _g = g;
    }
    [HttpGet]
    public async Task&lt;IActionResult&gt; Index() =&gt; Ok(await _g
         .V&lt;Airport&gt;()
         .ToArrayAsync());
    [HttpGet("/{airPortCode}")]
    public async Task&lt;IActionResult&gt; Single(string airPortCode)
    {
        var maybeAirport = await _g
            .V&lt;Airport&gt;()
            .Where(airport =&gt; airport.Code == airPortCode)
            .FirstOrDefaultAsync();
        return maybeAirport is { } airport
            ? Ok(airport)
            : NotFound();
    }
}</code></pre> 
</div> 
<h2>Gremlinq.Extensions</h2> 
<p>Beyond the open source core library, a set of commercial extensions is available on the <a href="https://docs.gremlinq.net/extensions/" rel="noopener noreferrer" target="_blank">Gremlinq website</a>, where they add frequently requested features:</p> 
<ul> 
 <li><strong>System.Text.Json deserialization</strong>: Use the modern .NET JSON serializer instead of Newtonsoft.Json</li> 
 <li><strong>OpenTelemetry instrumentation</strong>: Integrate graph database tracing into your observability stack</li> 
 <li><strong>Traversal strategies</strong>: Apply server-side traversal strategies to your queries</li> 
 <li><strong>Groovy script execution</strong>: Execute raw Gremlin scripts when needed for advanced scenarios</li> 
 <li><strong>Transactions</strong>: Transaction support (currently in development)</li> 
</ul> 
<p>These extensions are licensed separately and can be added to your project as needed.</p> 
<h2>Performance and trade-offs</h2> 
<p>Gremlinq builds syntactically correct Gremlin traversals and submits them to Neptune, so query execution and optimization ultimately happen in Neptune’s Gremlin engine. In many cases, the strongly typed DSL produces traversals that correspond closely to what you would write by hand, and the usual approaches to monitoring and tuning query performance in Neptune still apply.</p> 
<p>If you identify a performance issue that requires a traversal Gremlinq can’t express conveniently, you can fall back to raw Gremlin for that specific case while keeping the rest of your application in the strongly typed Gremlinq model. In those situations, it’s important to understand what Gremlinq is sending to Neptune so that you can compare the generated traversal with a hand‑tuned version and choose the approach that meets your performance and maintainability requirements.</p> 
<h3>Inspecting the generated Gremlin</h3> 
<p>If you want to see the underlying Gremlin query that Gremlinq is producing and sending to Neptune. You can use its <code>Debug()</code> method that shows the Gremlin traversal corresponding to a query, which you can log or run directly against Neptune when investigating performance issues:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Example: inspect the Gremlin for a query
var query = _g
    .V&lt;Airport&gt;()
    .Where(airport =&gt; airport.Code == "SEA")
    .Out&lt;Route&gt;()
    .OfType&lt;Airport&gt;();
// Get the raw Gremlin (or bytecode representation, depending on provider)
var debugInfo = query.Debug();
// Log or print for inspection
Console.WriteLine(debugInfo);</code></pre> 
</div> 
<p>By capturing this output, you can use the output in a variety of ways. It can be helpful in debugging because you might need the actual query sent to Neptune when trying to understand server logging. For example, the <a href="https://docs.aws.amazon.com/neptune/latest/userguide/slow-query-logs.html" rel="noopener noreferrer" target="_blank">Neptune slow query log</a> captures the Gremlin queries that Gremlinq generates, so being able to trace a query back to the Gremlinq code that produced it is essential for identifying improvement opportunities. It will also be helpful in using <a href="https://docs.aws.amazon.com/neptune/latest/userguide/gremlin-traversal-tuning.html" rel="noopener noreferrer" target="_blank">Neptune’s /profile or /explain APIs</a>, which requires you to send the Gremlin that Gremlinq generates to understand query execution.</p> 
<p>If a traversal that you want is difficult to express using the DSL, you can also fall back to lower-level Gremlin.NET traversals or other supported mechanisms in that specific part of your code base, while keeping the rest of the application in Gremlinq for type-safety and ergonomics.</p> 
<h2>Conclusion</h2> 
<p>Developing graph database applications with Gremlin can benefit from the type-safety and developer productivity features familiar from the .NET platform when using tools such as Gremlinq.</p> 
<p>For .NET development with Amazon Neptune, Gremlinq offers an option that brings strong typing and a LINQ-like development style to Gremlin-based graph applications. This approach introduces an additional abstraction layer between your code and the underlying Gremlin traversals but offers compile-time type checking and familiar C# patterns while requiring you to work within the constraints of the DSL. You can use the tooling to inspect the generated traversals and replace specific queries with hand-tuned Gremlin should the need arise. The project templates streamline your initial setup, and the <a href="https://docs.gremlinq.net" rel="noopener noreferrer" target="_blank">documentation</a> provides deeper exploration of projections, aggregations, groupings, and more advanced query patterns.</p> 
<p>Whether you’re building a recommendation engine, modeling complex organizational hierarchies, or implementing fraud detection logic, you can evaluate whether the type-safety and productivity benefits of working with strongly typed code align with your team’s priorities and development workflow.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Stephen Mallette" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/22/DBBLOG-5335-1.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Stephen Mallette</h3> 
  <p><a href="https://www.linkedin.com/in/stephen-mallette/" rel="noopener" target="_blank">Stephen</a> is a member of the Amazon Neptune team at AWS. He first started contributing to Apache TinkerPop, the home of the Gremlin graph query language, in 2009 and continues to share his experience there.</p> 
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Daniel Weber" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/22/DBBLOG-5335-2.png" width="120" />
  </div> 
  <h3 class="lb-h4">Daniel Weber</h3> 
  <p><a href="https://www.linkedin.com/in/danielcweber/" rel="noopener" target="_blank">Daniel</a> is a Senior Software Engineer at ExRam Innovations in Germany, where he created and maintains ExRam.Gremlinq. With over two decades of C# and .NET experience and a master’s degree in computer science from RWTH Aachen University, Daniel brings deep expertise in developer tooling for graph databases.</p> 
 </div> 
</footer>
