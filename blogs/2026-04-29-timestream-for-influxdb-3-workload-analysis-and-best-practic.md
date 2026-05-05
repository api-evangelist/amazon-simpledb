---
title: "Timestream for InfluxDB 3 workload analysis and best practices"
url: "https://aws.amazon.com/blogs/database/timestream-for-influxdb-3-workload-analysis-and-best-practices/"
date: "Wed, 29 Apr 2026 15:32:00 +0000"
author: "Forest Vey"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
<p>Selecting the right instance size for your <a href="https://aws.amazon.com/timestream/" rel="noopener noreferrer" target="_blank">Amazon Timestream for InfluxDB 3</a> deployment is one of the most impactful decisions you’ll make when architecting your time series infrastructure. An undersized instance can lead to degraded query performance and ingestion bottlenecks, while an oversized instance means paying for unused capacity.</p> 
<p>In this blog post we will walk you through on how to choose the sizing for your deployment. You can use this guide to help align your business use-case with AWS offerings for Timestream for InfluxDB 3. Visit <a href="https://docs.aws.amazon.com/timestream/latest/developerguide/influxdb3.html" rel="noopener noreferrer" target="_blank">our documentation</a> to get started using the Amazon Timestream managed InfluxDB 3 today.</p> 
<h2>Sizing guidance</h2> 
<h3>Understanding performance characteristics</h3> 
<p>Choosing the right instance size for your Amazon Timestream for InfluxDB 3 deployment requires understanding how the database performs under different workload patterns. </p> 
<p>Performance in time series databases is influenced by numerous interconnected factors. The complexity of your queries matters significantly as simple aggregations over recent data perform differently than complex analytical queries spanning months of historical data. Your data model plays a crucial role; high-cardinality datasets with millions of unique tag combinations behave differently than low-cardinality monitoring data. Write patterns affect performance as well, from the size of your batches to the rate of ingestion and the structure of your line protocol points.</p> 
<p>Concurrent operations create their own dynamics. When multiple applications query the database simultaneously while data continues to flow in, resources must be balanced between ingestion and query processing. Query time ranges are of significant impact as InfluxDB 3’s caching and in-memory optimizations make recent data queries exceptionally fast, while historical queries (in Enterprise clusters) use the compactor and Parquet storage for efficient retrieval.</p> 
<p>Our goal with this post isn’t to provide exact predictions for your workload, but rather to provide you with an informed starting point for your deployment.</p> 
<h3>Data and query considerations</h3> 
<p>When aiming to improve query and ingestion performance, consider the following:</p> 
<p><strong>If you have low cardinality</strong>&nbsp;(fewer unique tag combinations), you can potentially see improved performance, as queries need to filter through fewer distinct series. While cardinality is not a problem anymore for InfluxDB 3, this does not mean runaway cardinality has no performance impact. With runaway cardinality, query operations will potentially use more memory and compaction performance will be impacted.</p> 
<p><strong>If your queries are simple</strong>&nbsp;(single-field retrievals rather than aggregations), you can benefit from InfluxDB 3’s <a href="https://docs.influxdata.com/influxdb3/enterprise/admin/last-value-cache/" rel="noopener noreferrer" target="_blank">Last Value Cache</a> and see significantly faster response times. If your queries perform multi-hour or multi-day aggregation on raw data, query performance will decrease as the size of your data grows. Always <a href="https://docs.influxdata.com/influxdb3/enterprise/plugins/library/official/downsampler/" rel="noopener noreferrer" target="_blank">downsample</a> using the processing engine when you can and pre-format your data according to how you want query results presented.</p> 
<p><strong>If your write batches are larger</strong>&nbsp;(5,000 points per batch for medium to 4xlarge instances and 10,000+ points per batch for larger instances), you can achieve better write throughput as the overhead per request decreases.</p> 
<p><strong>If your queries target longer time ranges</strong>&nbsp;or historical data beyond the recent cache window, <a href="https://docs.influxdata.com/influxdb3/enterprise/plugins/library/official/downsampler/" rel="noopener noreferrer" target="_blank">downsampling</a>, for all editions, and Enterprise edition’s compactor can help maintain good performance.</p> 
<p><strong>If you have fewer concurrent readers</strong>, you can see better per-query performance as resources aren’t divided among as many simultaneous operations.</p> 
<h2>Monitoring your deployment</h2> 
<p>After deploying your instance, monitoring performance is crucial for validating your sizing decisions and identifying optimization opportunities. Amazon CloudWatch provides essential metrics for Timestream for InfluxDB 3 including CPU utilization and memory usage. The <a href="https://docs.influxdata.com/influxdb3/enterprise/plugins/library/official/system-metrics/?section=influxdb3%252Fenterprise%252Fplugins" rel="noopener noreferrer" target="_blank">System Metrics Plugin</a> included with InfluxDB 3’s processing engine collects server-level performance data similar to CloudWatch, including detailed CPU statistics (overall and per-core), memory usage breakdowns, disk I/O performance, and network interface statistics. For detailed, long-term metrics, the System Metrics Plugin is a better option compared to CloudWatch since CloudWatch logs metrics every 10 seconds with the same level of granularity while the System Metrics Plugin can be <a href="https://docs.influxdata.com/influxdb3/enterprise/reference/cli/influxdb3/create/trigger/?section=influxdb3%252Fenterprise%252Freference#create-a-trigger-with-a-schedule" rel="noopener noreferrer" target="_blank">configured to use a custom schedule</a>.</p> 
<p>For deeper insights into database-specific performance metrics, you can scrape the metrics endpoint to collect comprehensive internal metrics that help you closely monitor query performance, write throughput, and other service-level indicators. We provide a comprehensive metrics collection solution that scrapes the /metrics endpoint from your Timestream for InfluxDB instances. This solution deploys an <a href="https://aws.amazon.com/marketplace" rel="noopener noreferrer" target="_blank">Amazon EC2</a> instance running <a href="https://www.influxdata.com/time-series-platform/telegraf/" rel="noopener noreferrer" target="_blank">Telegraf</a> to continuously collect internal engine metrics such as query performance, write throughput, and memory usage patterns, then ingests them into CloudWatch for visualization in a pre-configured Grafana dashboard. The dashboard includes panels for monitoring key performance indicators aligned with instance sizing specifications, helping you validate your sizing decisions and identify optimization opportunities.</p> 
<p>For deployment instructions, configuration options, and example scripts, visit our <a href="https://github.com/awslabs/amazon-timestream-tools/tree/mainline/integrations/influxdb_metrics_dashboard" rel="noopener noreferrer" target="_blank">GitHub repository</a>. The repository includes a complete <a href="https://aws.amazon.com/cdk/" rel="noopener noreferrer" target="_blank">AWS CDK</a> application that automates the setup process, from Telegraf configuration to Grafana dashboard creation.</p> 
<h3>Getting started with sizing</h3> 
<p>When planning your initial deployment, we recommend this approach:</p> 
<ol> 
 <li><strong>Estimate your baseline requirements</strong>: Calculate your expected write rate (points per second) and query concurrency (simultaneous queries). Add additional headroom to accommodate growth and handle usage spikes.</li> 
 <li><strong>Choose a starting instance size</strong>:<br />Use the following as a guide for determining instance size in ideal conditions. Deploy an instance that roughly meets your needs, then adjust according to performance:<br /> 
  <table border="1px" cellpadding="10px"> 
   <tbody>
    <tr> 
     <td><strong>Writes (lines per second)</strong></td> 
     <td><strong>Reads (Queries per second)</strong></td> 
     <td><strong>Instance class</strong></td> 
    </tr> 
    <tr> 
     <td>~35,000</td> 
     <td>~40</td> 
     <td>db.influx.medium</td> 
    </tr> 
    <tr> 
     <td>&lt;100,000</td> 
     <td>~140</td> 
     <td>db.influx.large</td> 
    </tr> 
    <tr> 
     <td>~120,000</td> 
     <td>~290</td> 
     <td>db.influx.xlarge</td> 
    </tr> 
    <tr> 
     <td>~130,000</td> 
     <td>~400</td> 
     <td>db.influx.2xlarge</td> 
    </tr> 
    <tr> 
     <td>~140,000</td> 
     <td>~425</td> 
     <td>db.influx.4xlarge</td> 
    </tr> 
    <tr> 
     <td>&lt;200,000</td> 
     <td>&lt;430</td> 
     <td>db.influx.8xlarge</td> 
    </tr> 
    <tr> 
     <td>~200,000</td> 
     <td>&lt;430</td> 
     <td>db.influx.12xlarge</td> 
    </tr> 
    <tr> 
     <td>~210,000</td> 
     <td>~430</td> 
     <td>db.influx.16xlarge</td> 
    </tr> 
    <tr> 
     <td>~230,000</td> 
     <td>~430</td> 
     <td>db.influx.24xlarge</td> 
    </tr> 
   </tbody>
  </table> 
  <ul> 
   <li>For workloads focused on recent data monitoring (3-5 days), start with db.influx.xlarge</li> 
   <li>For higher write throughput or query concurrency needs, consider db.influx.2xlarge or db.influx.4xlarge</li> 
   <li>For maximum throughput requirements, evaluate db.influx.8xlarge or db.influx.24xlarge</li> 
   <li>If you need historical data retention and analysis, choose Enterprise edition with appropriate instance size</li> 
  </ul> </li> 
 <li>Create and configure a <a href="https://docs.aws.amazon.com/timestream/latest/developerguide/parameter-groups.html" rel="noopener noreferrer" target="_blank">parameter group</a> for your deployment according to your instance size and workload characteristics.</li> 
 <li><strong>Test with representative data</strong>: Load a dataset that matches your production cardinality and time range, then run your actual query patterns to validate performance.</li> 
 <li><strong>Monitor and adjust</strong>: Use CloudWatch metrics and the <a href="https://docs.aws.amazon.com/timestream/latest/developerguide/processing-engine.html#available-plugins" rel="noopener noreferrer" target="_blank">System Metrics Plugin</a> to track CPU, memory, query latency, and write throughput. Scale up if you consistently see high resource utilization or degraded performance. 
  <ul> 
   <li><strong>A healthy instance, operating at production levels, should have a CPU and memory profile between 40% and 70% on average.</strong></li> 
   <li><strong>If you are normally below 40% and your workload is stable, you are overprovisioning.</strong></li> 
   <li><strong>If you have spikes that breach 70% threshold, consider optimizations or scaling. Keep in mind that queries have a bigger impact on CPU usage than writes.</strong></li> 
  </ul> </li> 
</ol> 
<p>Remember that you can scale your instance size up or down as your workload evolves with brief interruption of downtime. Start conservatively, monitor carefully, and adjust based on real-world performance data from your specific use case.</p> 
<h2>Summary</h2> 
<p>Sizing your Amazon Timestream for InfluxDB 3 deployment effectively requires balancing throughput requirements, query patterns, and cost considerations. Use this blog post as a starting point for determining how Amazon Timestream for InfluxDB 3 can elevate your business needs. By analyzing your workload patterns, query complexity, and historical data needs, you can map your requirements to the appropriate configuration.</p> 
<p>Visit the <a href="https://docs.aws.amazon.com/timestream/latest/developerguide/influxdb3.html#what-is-timestream-for-influxdb-3" rel="noopener noreferrer" target="_blank">Amazon Timestream for InfluxDB documentation</a> today and start building your time series workflows with the power, flexibility, and scalability that your business demands.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Victor Servin" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/23/DBBLOG-5260-part2-1.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Victor Servin</h3> 
  <p><a href="https://www.linkedin.com/in/victorservin/" rel="noopener" target="_blank">Victor</a> is a Senior Product Manager for the Amazon Timestream team at AWS. With years of expertise in supporting startups with product-led growth strategies and scalable architecture, Victor’s data-driven approach is perfectly suited to drive the adoption of analytical products like Timestream. His extensive experience and commitment to customer success helps him support customers to efficiently achieve their goals.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Forest Vey" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/23/DBBLOG-5260-part2-2.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Forest Vey</h3> 
  <p><a href="https://www.linkedin.com/in/forest-vey-158712225/" rel="noopener" target="_blank">Forest</a> is a Team Lead at Improving. He is experienced working with time-series and serverless technologies. Forest has a growing passion for cloud development and embedded systems. When not immersed in learning about software development, he enjoys rock climbing and hiking with his friends.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Trevor Bonas" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/23/DBBLOG-5260-part2-3.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Trevor Bonas</h3> 
  <p><a href="https://www.linkedin.com/in/trevorbonas/" rel="noopener" target="_blank">Trevor</a> is a Senior Software Developer at Improving. Trevor has experience with time-series databases and ODBC driver development. In his free time, Trevor writes fiction and dabbles in game development.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Fred Park" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/23/DBBLOG-5260-part2-4.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Fred Park</h3> 
  <p>Fred is a Senior Software Developer at Improving, specializing in time series databases and observability for AI agents. He’s drawn to emerging technology, with a background that ranges from VR and cloud services to quantum computing. When he’s not coding, you’ll find him on a trail or making music with his friends.</p>
 </div> 
</footer>
