---
title: "DSQL SQL Dialect: How Amazon Aurora DSQL differs from single-instance PostgreSQL"
url: "https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/"
date: "Mon, 27 Apr 2026 17:43:43 +0000"
author: "Rob Petersen"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
<p><a href="https://aws.amazon.com/rds/aurora/dsql/" rel="noopener" target="_blank">Amazon Aurora DSQL</a> is based on open source PostgreSQL (today v16) but is also a distributed database. This means that there are some differences in supported features and behavior. Understanding the differences can help you save time and take advantage of the unique capabilities of Amazon Aurora DSQL.</p> 
<p>Knowing exactly where Amazon Aurora DSQL aligns with standard PostgreSQL and where it diverges helps you to reduce risk and design schemas that perform well from day one. You might find that most existing PostgreSQL applications work with minimal changes.</p> 
<p>This post is for database architects, developers, and DBAs who must evaluate Amazon Aurora DSQL or work with PostgreSQL workloads on a distributed database.</p> 
<h2>What’s the same</h2> 
<p>The short answer: most of it. Today, Amazon Aurora DSQL uses standard PostgreSQL v16 and the v3.2+ <a href="https://www.postgresql.org/docs/current/protocol.html" rel="noopener" target="_blank">wire protocol</a>. Standard PostgreSQL clients, drivers, and ORMs: psql, pgjdbc, psycopg, Django, ActiveRecord, Hibernate, connect and work as expected. Your SQL queries, assuming they use supported features, return identical results: same NULL handling, sort order, numeric precision, and string behavior.</p> 
<p>The core SQL surface area is intact:</p> 
<ul> 
 <li><strong>Standard DML: </strong>SELECT (with joins, subqueries, aggregations), INSERT, UPDATE, DELETE</li> 
 <li><strong>DDL: </strong>CREATE TABLE, ALTER TABLE, CREATE VIEW, DROP VIEW</li> 
 <li><strong>Transaction control: </strong>BEGIN, COMMIT, ROLLBACK</li> 
 <li><strong>Core data types: </strong>numeric types, character types, date/time types, Boolean, bytea</li> 
</ul> 
<p>Some PostgreSQL features aren’t available in Amazon Aurora DSQL. For the complete list of supported and unsupported features, see the&nbsp;<a href="https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility.html" rel="noopener" target="_blank">Aurora DSQL and PostgreSQL Compatibility</a>&nbsp;documentation. The feature set evolves as Amazon Aurora DSQL adds support.</p> 
<p>The key point: if your application uses standard PostgreSQL SQL for transactional workloads, the dialect compatibility is high. The differences that matter are architectural, not syntactic.</p> 
<h2>What’s different and why</h2> 
<p>The dialect differences in Amazon Aurora DSQL stem from its distributed, shared-nothing architecture. Understanding the architecture explains the differences, which makes them predictable rather than surprising.</p> 
<h3>Storage is ordered by primary key</h3> 
<p>This is the most fundamental difference, and it shapes everything else.</p> 
<p>In PostgreSQL, tables use a&nbsp;<em>heap</em>: rows are stored in unordered pages with no relationship to the primary key. You can run&nbsp;<strong>CLUSTER</strong>&nbsp;to reorder a table by an index once, but PostgreSQL doesn’t maintain that order as data changes.</p> 
<p>In Amazon Aurora DSQL, data is&nbsp;<em>stored and maintained in primary key order</em>. Think of it as a table that’s continuously&nbsp;<strong>Clustered</strong>. This applies to both tables and secondary indexes (which are ordered by their key columns).</p> 
<h4>What this means for the SQL that you write:</h4> 
<ul> 
 <li>Range scans on the primary key are physically sequential reads. Queries that filter on primary key ranges are inherently efficient.</li> 
 <li>Primary key choice has a much larger impact than in PostgreSQL. In a heap, a bad primary key is compensated by indexes. In Amazon Aurora DSQL, the primary key is the physical layout.</li> 
 <li>Sequential or monotonically increasing keys (auto-increment, timestamps) create hot spots in a distributed system. Randomly distributed keys (UUIDs) spread load. The same principle applies to secondary indexes. Their key order determines physical layout and scan efficiency.</li> 
</ul> 
<h4>Not all operations push down to storage</h4> 
<p>Amazon Aurora DSQL separates compute and storage. This is one of the keys to its ability to automatically scale and deliver a fully serverless operation. When a query runs, the critical performance question is: can the filter or operation be evaluated at the storage layer, or does the compute layer have to fetch rows and evaluate them locally?</p> 
<p>This affects the dialect in two concrete ways:</p> 
<p><strong>Index key type restrictions.</strong>&nbsp;Not every PostgreSQL data type can be used as the key of an index in Amazon Aurora DSQL. If you try to create an index on an unsupported key type, you will get an error. Check the&nbsp;<a href="https://docs.aws.amazon.com/aurora-dsql/latest/userguide/" rel="noopener" target="_blank">current documentation</a>&nbsp;for the supported set as this evolves over time.</p> 
<p><strong>Operation push down.</strong>&nbsp;Straightforward equality and range comparisons on supported types generally push down to storage. Complex expressions, function calls, or operations on types that the storage layer doesn’t natively handle are evaluated at the compute layer after fetching rows. This doesn’t change your SQL syntax but alters performance characteristics. Use&nbsp;<strong>EXPLAIN</strong>&nbsp;to identify filters that aren’t being pushed down.</p> 
<p><strong>Covering indexes matter more here.</strong>&nbsp;Because of the compute/storage separation, a query that can be answered entirely from an index (without fetching the base table row) avoids an extra round trip to storage. Use&nbsp;<strong>INCLUDE</strong>&nbsp;columns aggressively:</p> 
<div class="hide-language"> 
 <pre><code class="lang-sql">-- Covers queries that filter on customer_id and return order_total, status

CREATE INDEX idx_orders_customer ON orders (customer_id) INCLUDE (order_total, status);</code></pre> 
</div> 
<h3> Optimistic concurrency control</h3> 
<p>PostgreSQL uses lock-based concurrency: transactions acquire row-level locks and block each other on conflict. Amazon Aurora DSQL uses&nbsp;<em>optimistic concurrency control (OCC):</em> transactions execute without locks and are checked for conflicts at commit time.</p> 
<p>This doesn’t change your SQL syntax, but it changes the dialect’s&nbsp;behavior:</p> 
<ul> 
 <li><strong>No deadlocks.</strong>&nbsp;Transactions don’t block each other.</li> 
 <li><strong>Serialization errors on conflict.</strong>&nbsp;When two transactions modify the same data, the one with the earlier commit timestamp wins. The other receives SQLSTATE&nbsp;<strong>40001</strong>. Your application must retry the entire transaction.</li> 
 <li><strong>Read-only transactions are conflict-free.</strong>&nbsp;They have zero commit latency and aren’t rejected.</li> 
 <li><strong>Isolation level is equivalent to PostgreSQL Repeatable Read.</strong>&nbsp;You don’t choose an isolation level, this is what you get.</li> 
</ul> 
<p>The practical dialect difference: <strong>SELECT … FOR UPDATE</strong> is syntactically supported, but it doesn’t block. Instead, conflicts are detected at commit time and result in an OCC serialization error. If your PostgreSQL application relies on <strong>SELECT … FOR UPDATE</strong> to serialize access to contested rows, expect those paths to surface as retry-able conflicts rather than blocking waits. Design for low contention and use idempotent transactions, that can be retried without additional business logic / new decisions.</p> 
<h3>Asynchronous DDL</h3> 
<p>In PostgreSQL, DDL is synchronous: when&nbsp;<strong>CREATE TABLE</strong>&nbsp;returns, the table exists. In Amazon Aurora DSQL, some DDL is synchronous and some is not. Straightforward operations like <strong>CREATE TABLE</strong> return synchronously, but DDL that requires background work such as [All CREATE INDEX operations must be CREATE INDEX ASYNC].</p> 
<p>Dialect constraints that follow from this:</p> 
<ul> 
 <li>Only&nbsp;one DDL statement per transaction.</li> 
 <li>DDL and DML cannot be mixed&nbsp;in the same transaction.</li> 
 <li>For asynchronous DDL, you must&nbsp;verify DDL completion&nbsp;(select * from sys.jobs; or wait for the job_id to finish) before depending on the schema change.</li> 
</ul> 
<p>Reads and writes continue uninterrupted during DDL operations.</p> 
<h3>Authentication is IAM, not passwords</h3> 
<p>Amazon Aurora DSQL replaces the PostgreSQL&nbsp;<strong>pg_hba.conf</strong>&nbsp;and username/password authentication with <a href="https://aws.amazon.com/iam/" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a>. You connect using short-lived tokens generated using the <a href="https://builder.aws.com/build/tools" rel="noopener" target="_blank">AWS SDK</a>. This doesn’t change the SQL dialect itself, but it changes every connection string and authentication flow in your application. For more information, see the&nbsp;<a href="https://docs.aws.amazon.com/aurora-dsql/latest/userguide/using-database-and-iam-roles.html" rel="noopener" target="_blank">IAM authentication guide.</a> For comprehensive access control patterns and security best practices, see <a href="https://aws.amazon.com/blogs/database/securing-amazon-aurora-dsql-access-control-best-practices/" rel="noopener" target="_blank">Securing Amazon Aurora DSQL: Access Control Best Practices</a>.</p> 
<h2>Unsupported features that affect the dialect</h2> 
<p>Some PostgreSQL features aren’t available in Amazon Aurora DSQL. Rather than list everything (always consult the&nbsp;<a href="https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility.html" rel="noopener" target="_blank">Aurora DSQL and PostgreSQL Compatibility</a>&nbsp;documentation), here are the categories that most commonly affect the way SQL is written: </p> 
<table border="1px" cellpadding="10px"> 
 <tbody>
  <tr> 
   <td><strong>Category</strong></td> 
   <td><strong>What’s unsupported</strong></td> 
   <td><strong>Workaround</strong></td> 
  </tr> 
  <tr> 
   <td>Types</td> 
   <td>Geometric types, pgvector</td> 
   <td>Not currently supported</td> 
  </tr> 
  <tr> 
   <td>Temporary tables</td> 
   <td>CREATE TEMP TABLE</td> 
   <td>Use CTEs, subqueries, or regular tables with cleanup</td> 
  </tr> 
  <tr> 
   <td>Procedural logic</td> 
   <td>PL/pgSQL, triggers</td> 
   <td>Move to application tier, Amazon EventBridge, or AWS Lambda</td> 
  </tr> 
  <tr> 
   <td>Extensions</td> 
   <td>PostgreSQL extensions</td> 
   <td>Not currently supported</td> 
  </tr> 
  <tr> 
   <td>Constraints</td> 
   <td>Foreign keys</td> 
   <td>Enforce referential integrity in application code</td> 
  </tr> 
  <tr> 
   <td>Column types</td> 
   <td>JSONB/JSON as column types</td> 
   <td>Store as TEXT, cast to JSON/JSONB at query runtime</td> 
  </tr> 
  <tr> 
   <td>Column types</td> 
   <td>Array column definitions</td> 
   <td>Use array functions</td> 
  </tr> 
  <tr> 
   <td>Cascading operations</td> 
   <td>ON DELETE CASCADE</td> 
   <td>Use soft deletes with deleted_at timestamp</td> 
  </tr> 
  <tr> 
   <td>Auto-increment</td> 
   <td>SERIAL / BIGSERIAL</td> 
   <td>Use GENERATED ALWAYS AS IDENTITY or explicit CREATE SEQUENCE with high cache size (≥65536)</td> 
  </tr> 
 </tbody>
</table> 
<p>These are the items most likely to require SQL changes during migration. The full list evolves as features are added, so we recommend that you verify against the current <a href="https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility.html" rel="noopener" target="_blank">documentation</a>.</p> 
<h2>Conclusion</h2> 
<p>Amazon Aurora DSQL uses a PostgreSQL parser, planner, and type system, so the SQL dialect is largely compatible. This post explores the architectural differences that matter and how they shape the way you write and think about SQL in a distributed database: key-ordered storage, optimistic concurrency control, asynchronous DDL, and IAM authentication.</p> 
<p>The focus is on understanding why Amazon Aurora DSQL behaves differently. Primary keys determine physical layout and write distribution, indexes are key-ordered structures that benefit from covering columns, transactions are optimistic and conflict-free for reads, and operations have limits designed for consistent performance in a distributed system. We recommend that you keep an eye on this service. Amazon Aurora DSQL is closing feature gaps (PG Sequences just launched!) and the differences between Amazon Aurora DSQL and standard PostgreSQL will continue to shrink.</p> 
<p>Create your first Aurora DSQL cluster today and experience distributed PostgreSQL with active-active writes across multiple AWS Regions. Amazon Aurora DSQL offers a <a href="https://aws.amazon.com/rds/aurora/dsql/pricing/" rel="noopener" target="_blank">free tier</a> with 100,000 DPUs per month and 1GB Storage.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Rob Petersen" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/22/db5378a1.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Rob Petersen</h3> 
  <p><a href="https://www.linkedin.com/in/rob-petersen-5532455b/" rel="noopener" target="_blank">Rob</a> is a Senior Technical Account Manager at AWS, bringing 20 years of IT industry experience to help customers accelerate their cloud adoption journey. His experience spans both leading large-scale cloud migrations and managing hybrid infrastructure operations, giving him unique insights into the challenges and opportunities organizations face during cloud adoption.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Alex Pawvathil" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/22/db5378a2.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Alex Pawvathil</h3> 
  <p><a href="https://www.linkedin.com/in/alex-p-578849a8/" rel="noopener" target="_blank">Alex</a> is a Senior Technical Account Manager at AWS specializing in database architecture and enterprise-scale implementations. With over 14 years of hands-on experience in cloud architecture, database strategy, and enterprise advisory, he is a go-to expert on Amazon RDS for SQL Server implementations and enterprise-scale deployments.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Brent Durst" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/22/db5378a3.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Brent Durst</h3> 
  <p><a href="https://www.linkedin.com/in/brent-durst-82aa72/" rel="noopener" target="_blank">Brent</a> is a Technical Account Manager for AWS Energy and Utilities Industries, where he serves as a strategic technical guide and advocate for his customers within AWS. With over 20 years of experience spanning roles in Program Management, Data and Operations, Solution Architecture, and FinOps, he brings a well-rounded perspective to every engagement.</p>
 </div> 
</footer>
