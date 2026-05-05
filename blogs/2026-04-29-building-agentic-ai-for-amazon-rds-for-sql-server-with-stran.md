---
title: "Building agentic AI for Amazon RDS for SQL Server with Strands and AgentCore"
url: "https://aws.amazon.com/blogs/database/building-agentic-ai-for-amazon-rds-for-sql-server-with-strands-and-agentcore/"
date: "Wed, 29 Apr 2026 15:27:07 +0000"
author: "Sudhir Amin"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
<p>If you manage <a href="https://aws.amazon.com/rds/sqlserver/" rel="noopener noreferrer" target="_blank">Amazon Relational Database Service (Amazon RDS) for SQL Server</a> instances, you’ve likely accumulated a collection of diagnostic scripts over the years. These scripts query for blocking sessions, identify slow-running procedures, monitor disk space, and analyze index usage. They represent expertise you’ve developed through years of troubleshooting database issues, often during critical late-night incidents. Your scripts can now form the foundation for AI agents that work continuously. <a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html" rel="noopener noreferrer" target="_blank">Amazon Bedrock AgentCore Runtime</a> with <a href="https://github.com/awslabs/strands-agents" rel="noopener noreferrer" target="_blank">Strands Agents</a> enables you to transform your existing T-SQL knowledge into an autonomous database management system. When you combine <a href="https://aws.amazon.com/what-is/large-language-model/" rel="noopener noreferrer" target="_blank">AI models</a> with purpose-built AI agents, you create agents that understand context, make informed decisions, and execute complex operations autonomously.</p> 
<p>In this post, we walk through building an agent that investigates blocking and deadlocks on Amazon RDS for SQL Server — two issues that directly impact application performance, cause transaction failures, and lead to user-facing timeouts. Using the Strands Agents framework, we convert the T-SQL queries DBAs already use for these investigations into agent tools, combine them into a single agent, and deploy it to AgentCore Runtime.</p> 
<p>We start by explaining what agents and tools are, then walk through common DBA scenarios and how tools address them. From there, we build the tools, define the agent with a system prompt, and deploy it to AgentCore Runtime.</p> 
<h2>What are agents?</h2> 
<p>Agents are software systems that use AI to reason, plan, and complete tasks on behalf of humans or systems. In the context of database operations, an agent receives a question or an alarm payload, decides which tools to call, executes them, interprets the results, and responds with a diagnosis. For more on the Strands Agents framework, see <a href="https://aws.amazon.com/blogs/opensource/introducing-strands-agents-an-open-source-ai-agents-sdk/" rel="noopener noreferrer" target="_blank">Introducing Strands Agents: An Open Source AI Agents SDK</a>.</p> 
<p>At the core of every Strands agent is the <a href="https://strandsagents.com/docs/user-guide/concepts/agents/agent-loop/" rel="noopener noreferrer" target="_blank">agent loop</a>: invoke the AI service, check if it wants to use a tool, execute the tool, then invoke the service again with the result. This cycle repeats until the model produces a final response.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">Input → [Reasoning (LLM) → Tool Selection → Tool Execution] → Response<br />              ↑_________________________________↓</code></pre> 
</div> 
<p>Each iteration adds to the conversation history. The agent tracks not just the original request, but every tool it has called and every result it has received. This accumulated context enables multi-step reasoning. For example, the agent can check for deadlocks, then analyze blocking chains, then correlate both findings into a single diagnosis.</p> 
<h2>What are tools?</h2> 
<p>Now that we understand how agents reason, let’s look at the building blocks they use. In agentic AI systems, tools are functions that an agent can invoke to interact with external systems, retrieve data, or perform actions. Tools enable the agent’s capabilities beyond text generation: querying databases, calling APIs, monitoring systems, and sending notifications. They are the bridge between an agent’s reasoning capabilities and the real world.</p> 
<p>In Strands, a tool is a Python function decorated with @tool. The function’s docstring tells the agent when and why to call it. The agent reads the docstring, matches it against the user’s question, and decides whether to invoke the function. This means that the existing diagnostic script that a DBA already uses can become a tool by wrapping it in a function with a @tool decorator and a descriptive docstring.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">from strands import Agent, tool<br /><br />@tool<br />def get_server_version() -&gt; str:<br />    """Return the SQL Server version. Use this to verify connectivity<br />    and identify the engine version."""<br />    conn = get_db_connection()<br />    cursor = conn.cursor()<br />    cursor.execute("SELECT @@VERSION")<br />    version = cursor.fetchone()[0]<br />    conn.close()<br />    return version</code></pre> 
</div> 
<p>The @tool decorator registers the function with the agent. The type hints define the input and output schema. The docstring is what the agent uses to decide when to call it. Write it as you would a runbook entry.</p> 
<h2>Common DBA scenarios</h2> 
<p>With agents and tools defined, let’s look at the specific database problems we want to solve.</p> 
<p><a href="https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-deadlocks-guide?view=sql-server-ver17" rel="noopener noreferrer" target="_blank"><strong>Deadlocks</strong></a><strong>.</strong> When a deadlock occurs, you need to identify the sessions involved, the SQL statements each was executing, the lock types and objects in contention, and the victim session. On Amazon RDS for SQL Server, deadlock information is captured in two ways:</p> 
<ul> 
 <li><strong>Trace flags 1204 and 1222</strong> — When enabled through a custom DB parameter group, these write deadlock details to the SQL Server error log, which can be published to Amazon CloudWatch Logs. See <a href="https://aws.amazon.com/blogs/database/monitor-deadlocks-in-amazon-rds-for-sql-server-and-set-notifications-using-amazon-cloudwatch/" rel="noopener noreferrer" target="_blank">Monitor deadlocks in Amazon RDS for SQL Server</a> for setup steps.</li> 
 <li><strong>The system_health extended event session</strong> — This built-in session captures deadlock graphs via the xml_deadlock_report event by default. Both sides of the deadlock, including SQL statements, lock types, and objects, are recorded in the XE file targets at D:\rdsdbdata\log.</li> 
</ul> 
<p><a href="https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/performance/understand-resolve-blocking" rel="noopener noreferrer" target="_blank"><strong>Blocking</strong></a><strong>.</strong> When applications report timeouts, you need to find the head blocker — the session at the root of the blocking chain — along with its SQL statement, the number of sessions waiting behind it, and how long they’ve been waiting. Capturing this information on RDS for SQL Server depends on what level of detail is needed:</p> 
<ul> 
 <li><strong>Current blocking</strong> — The DMVs sys.dm_exec_requests and sys.dm_exec_sql_text provide the current blocking state, including blocker and blocked session IDs, wait types, and SQL text. This data remains available without additional configuration.</li> 
 <li><strong>Historical blocking (blocked session only)</strong> — The system_health session captures lock waits exceeding 30 seconds via the wait_info event. This records the blocked session’s ID and SQL text, but not the blocker’s identity. CloudWatch Logs does not contain system_health data — it only receives the SQL Server error log.</li> 
 <li><strong>Historical blocking (both sides)</strong> — To capture both the blocker and blocked session after the fact, set blocked process threshold in your DB parameter group and create a custom extended event session for blocked_process_report. The system_health session does not capture this event. See <a href="https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.

ExtendedEvents.html" rel="noopener noreferrer" target="_blank">Using extended events with Amazon RDS for Microsoft SQL Server</a> for configuration details.</li> 
</ul> 
<h2>How tools address these scenarios</h2> 
<p>Now let’s map each scenario to a tool that the agent can call.</p> 
<table border="1px" cellpadding="10px"> 
 <thead> 
  <tr> 
   <th>Tool</th> 
   <th>What it does</th> 
   <th>Data source</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>get_deadlock_graphs</td> 
   <td>Reads deadlock graphs with full details for both sides</td> 
   <td>system_health Extended Event (XE) files (xml_deadlock_report)</td> 
  </tr> 
  <tr> 
   <td>get_blocking_chains</td> 
   <td>Walks the current blocking hierarchy, identifies head blockers and their SQL</td> 
   <td>DMVs (sys.dm_exec_requests, sys.dm_exec_sql_text)</td> 
  </tr> 
  <tr> 
   <td>get_session_details</td> 
   <td>Retrieves login, host, program, and SQL for a specific session</td> 
   <td>DMV (sys.dm_exec_sessions)</td> 
  </tr> 
  <tr> 
   <td>send_diagnostic_report</td> 
   <td>Sends the agent’s findings and recommendations to the DBA team</td> 
   <td><a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon SNS</a></td> 
  </tr> 
  <tr> 
   <td>get_blocked_process_reports</td> 
   <td>Reads historical blocking details including both blocker and blocked sessions</td> 
   <td>Custom XE session (blocked_process_report)</td> 
  </tr> 
 </tbody> 
</table> 
<h2>Tool code</h2> 
<p>Let’s see how these tools translate into Python code. Each tool wraps an existing SQL Server diagnostic query using the Strands @tool decorator.</p> 
<p>The tools share a connection helper that retrieves credentials from <a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a>:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">from strands import Agent, tool<br />import boto3, pymssql, json, os<br />from datetime import datetime, timedelta<br /><br />def get_db_connection():<br />    client = boto3.client('secretsmanager', region_name=os.getenv('AWS_REGION', 'us-west-2'))<br />    secret = client.get_secret_value(SecretId=os.getenv('DB_SECRET_ID'))<br />    creds = json.loads(secret['SecretString'])<br />    return pymssql.connect(<br />        server=creds['host'], user=creds['username'],<br />        password=creds['password'], port=creds['port']<br />    )</code></pre> 
</div> 
<p><strong>Deadlock detection.</strong> Reads deadlock graphs from the system_health XE files:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">@tool<br />def get_deadlock_graphs(hours: int = 24) -&gt; dict:<br />    """Read deadlock graphs from the system_health extended event session.<br />    Returns XML deadlock graphs with process lists, SQL statements, and lock<br />    details for both sides of each deadlock. Use this when applications report<br />    error 1205, or for periodic deadlock analysis."""<br />    conn = get_db_connection()<br />    cursor = conn.cursor(as_dict=True)<br />    cursor.execute("""<br />        SELECT<br />            CAST(event_data AS XML).value('(event/@timestamp)[1]', 'DATETIME2') AS event_time,<br />            CAST(event_data AS XML).value('(event/data[@name="xml_report"]/value)[1]',<br />                'NVARCHAR(MAX)') AS deadlock_graph<br />        FROM sys.fn_xe_file_target_read_file(<br />            'd:\\rdsdbdata\\log\\system_health*.xel', NULL, NULL, NULL)<br />        WHERE CAST(event_data AS XML).value('(event/@name)[1]', 'VARCHAR(100)')<br />              = 'xml_deadlock_report'<br />          AND CAST(event_data AS XML).value('(event/@timestamp)[1]', 'DATETIME2')<br />              &gt; DATEADD(HOUR, -%s, GETUTCDATE())<br />        ORDER BY event_time DESC<br />    """, (hours,))<br />    results = cursor.fetchall()<br />    conn.close()<br />    return {"deadlock_count": len(results), "deadlock_graphs": results}</code></pre> 
</div> 
<p><strong>Blocking chain analysis.</strong> Walks the blocking hierarchy using a recursive CTE:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">@tool<br />def get_blocking_chains() -&gt; dict:<br />    """Walk the current blocking chain hierarchy using sys.dm_exec_requests<br />    and sys.dm_exec_sql_text. Returns head blockers, their SQL, wait types,<br />    durations, and all downstream blocked sessions. Use this when applications<br />    report timeouts or hangs."""<br />    conn = get_db_connection()<br />    cursor = conn.cursor(as_dict=True)<br />    cursor.execute("""<br />        WITH BlockingChain AS (<br />            SELECT r.session_id, r.blocking_session_id, r.wait_type,<br />                   r.wait_time / 1000.0 AS wait_seconds, r.status, r.command,<br />                   t.text AS sql_text, DB_NAME(r.database_id) AS database_name,<br />                   0 AS chain_level<br />            FROM sys.dm_exec_requests r<br />            CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t<br />            WHERE r.blocking_session_id = 0<br />              AND r.session_id IN (<br />                  SELECT DISTINCT blocking_session_id FROM sys.dm_exec_requests<br />                  WHERE blocking_session_id != 0)<br />            UNION ALL<br />            SELECT r.session_id, r.blocking_session_id, r.wait_type,<br />                   r.wait_time / 1000.0, r.status, r.command, t.text,<br />                   DB_NAME(r.database_id), bc.chain_level + 1<br />            FROM sys.dm_exec_requests r<br />            CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t<br />            JOIN BlockingChain bc ON r.blocking_session_id = bc.session_id<br />        )<br />        SELECT session_id, blocking_session_id, wait_type, wait_seconds,<br />               status, command, sql_text, database_name, chain_level<br />        FROM BlockingChain ORDER BY chain_level, wait_seconds DESC<br />    """)<br />    results = cursor.fetchall()<br />    conn.close()<br />    head_blockers = [r for r in results if r['chain_level'] == 0]<br />    return {"total_blocked_sessions": len([r for r in results if r['chain_level'] &gt; 0]),<br />            "head_blockers": len(head_blockers), "blocking_chains": results}</code></pre> 
</div> 
<p><strong>Session details.</strong> Retrieves context for a specific session:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">@tool<br />def get_session_details(session_id: int) -&gt; dict:<br />    """Get login name, host, program, and current SQL for a specific session.<br />    Use this after identifying a head blocker."""<br />    conn = get_db_connection()<br />    cursor = conn.cursor(as_dict=True)<br />    cursor.execute("""<br />        SELECT s.session_id, s.login_name, s.host_name, s.program_name,<br />               s.status, s.transaction_isolation_level,<br />               s.last_request_start_time, s.last_request_end_time,<br />               t.text AS current_sql<br />        FROM sys.dm_exec_sessions s<br />        OUTER APPLY sys.dm_exec_sql_text(s.most_recent_sql_handle) t<br />        WHERE s.session_id = %s<br />    """, (session_id,))<br />    result = cursor.fetchone()<br />    conn.close()<br />    return {"session_id": session_id, "details": result}</code></pre> 
</div> 
<p><strong>Diagnostic reporting.</strong> Sends findings via SNS:</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">@tool<br />def send_diagnostic_report(subject: str, report: str) -&gt; dict:<br />    """Send a diagnostic report via SNS email to the DBA team. Use this after<br />    completing an investigation to deliver findings and recommendations."""<br />    sns = boto3.client('sns', region_name=os.getenv('AWS_REGION', 'us-west-2'))<br />    topics = sns.list_topics()['Topics']<br />    topic_arn = next((t['TopicArn'] for t in topics<br />                      if os.getenv('SNS_TOPIC_NAME') in t['TopicArn']), None)<br />    if topic_arn:<br />        sns.publish(TopicArn=topic_arn, Subject=subject[:100], Message=report)<br />        return {"status": "sent", "topic": topic_arn}<br />    return {"status": "no_topic_found"}</code></pre> 
</div> 
<p><strong>Historical blocked process reports.</strong> Reads blocker and blocked session details from a custom XE session. Requires blocked process threshold set in the DB parameter group and a custom extended event session capturing blocked_process_report with a file target. See <a href="https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.

ExtendedEvents.html" rel="noopener noreferrer" target="_blank">Using extended events with Amazon RDS for Microsoft SQL Server</a> for session creation.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python">@tool<br />def get_blocked_process_reports(hours: int = 24) -&gt; dict:<br />    """Read blocked process reports from extended event file targets.<br />    Returns both blocker and blocked session details including SQL text,<br />    wait time, and lock resources. Requires a custom XE session capturing<br />    blocked_process_report. Use this for historical blocking analysis<br />    when you need to identify who was holding the lock."""<br />    conn = get_db_connection()<br />    cursor = conn.cursor(as_dict=True)<br />    cursor.execute("""<br />        SELECT<br />            CAST(event_data AS XML).value('(event/@timestamp)[1]', 'DATETIME2') AS event_time,<br />            CAST(event_data AS XML).value('(event/data[@name="duration"]/value)[1]', 'BIGINT') AS duration_ms,<br />            CAST(event_data AS XML).value('(event/data[@name="blocked_process"]/value)[1]',<br />                'NVARCHAR(MAX)') AS blocked_process_report<br />        FROM sys.fn_xe_file_target_read_file(<br />            'd:\\rdsdbdata\\log\\blocked*.xel', NULL, NULL, NULL)<br />        WHERE CAST(event_data AS XML).value('(event/@timestamp)[1]', 'DATETIME2')<br />              &gt; DATEADD(HOUR, -%s, GETUTCDATE())<br />        ORDER BY event_time DESC<br />    """, (hours,))<br />    results = cursor.fetchall()<br />    conn.close()<br />    return {"blocked_process_count": len(results), "reports": results}</code></pre> 
</div> 
<p><strong>Note:</strong> All code examples use parameterized queries (%s placeholders) to help prevent SQL injection.</p> 
<h2>Defining the agent: System prompt and tools</h2> 
<p>With the tools built, we now define the agent itself. The <a href="https://strandsagents.com/docs/user-guide/concepts/agents/prompts/" rel="noopener noreferrer" target="_blank">system prompt</a> defines how the agent reasons about problems, and the tools list tells it what capabilities are available.</p> 
<div class="hide-language"> 
 <pre><code class="lang-python"># ---------------------------------------------------------------------------
# AgentCore app and model configuration
# ---------------------------------------------------------------------------
from strands import Agent, tool<br />from strands.models import BedrockModel<br />from bedrock_agentcore.runtime import BedrockAgentCoreApp<br />import boto3, pymssql, json, os<br />from datetime import datetime, timedelta<br /><br />app = BedrockAgentCoreApp()<br /><br />model = BedrockModel(<br />    model_id=os.getenv('BEDROCK_MODEL_ID', '&lt;your-model-id&gt;'),<br />    region_name=os.getenv('AWS_REGION', '&lt;your-region&gt;'),<br />    temperature=0.3<br />)<br />
# ---------------------------------------------------------------------------
# Database connection helper — retrieves credentials from Secrets Manager
# ---------------------------------------------------------------------------
<br />def get_db_connection():<br />    client = boto3.client('secretsmanager', region_name=os.getenv('AWS_REGION', '&lt;your-region&gt;'))<br />    secret = client.get_secret_value(SecretId=os.getenv('DB_SECRET_ID'))<br />    creds = json.loads(secret['SecretString'])<br />    return pymssql.connect(<br />        server=creds['host'], user=creds['username'],<br />        password=creds['password'], port=creds['port']<br />    )<br /><br /># --- Paste tool code here ---<br /># @tool get_deadlock_graphs       (from Deadlock detection section)<br /># @tool get_blocking_chains       (from Blocking chain analysis section)<br /># @tool get_session_details       (from Session details section)<br /># @tool get_blocked_process_reports (from Historical blocked process section)<br /># @tool send_diagnostic_report    (from Diagnostic reporting section)<br /><br /># ---------------------------------------------------------------------------
# Agent definition — system prompt guides reasoning, tools define capabilities
# ---------------------------------------------------------------------------
agent = Agent(<br />    system_prompt="""You are a SQL Server DBOps Agent for Amazon RDS.<br />When investigating issues:<br />1. Start with the symptoms — don't run every tool on every question<br />2. Use the deadlock tool when applications report error 1205 or transaction failures<br />3. Use blocking chain tools when applications report timeouts or lock waits<br />4. Use blocked process reports for historical blocking analysis when current blocking has resolved<br />5. Correlate findings across data sources before giving your assessment<br />6. Provide severity (Critical/Warning/Info) and specific, actionable recommendations<br />7. When sending diagnostic reports via SNS, include: affected session IDs, SQL statements, wait types, duration, host/IP, login name, root cause, and specific remediation steps""",<br />    model=model,<br />    tools=[get_deadlock_graphs, get_blocking_chains, get_session_details,<br />           get_blocked_process_reports, send_diagnostic_report]<br />)<br /># ---------------------------------------------------------------------------
# AgentCore entrypoint — receives payload from agentcore invoke
# ---------------------------------------------------------------------------
<br />@app.entrypoint<br />def handler(payload):<br />    response = agent(payload.get("prompt", ""))<br />    return response.message['content'][0]['text']<br /><br />if __name__ == "__main__":<br />    app.run()</code></pre> 
</div> 
<h2></h2> 
<h2>Solution workflow </h2> 
<p>At the heart of this transformation lies the integration of three key technologies:</p> 
<ul> 
 <li><a href="https://strandsagents.com/" rel="noopener noreferrer" target="_blank"><strong>Strands agents &amp; tools</strong></a> — Wrap existing T-SQL expertise in intelligent agents that reason about database conditions and execute responses automatically.</li> 
 <li><a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime.html" rel="noopener noreferrer" target="_blank"><strong>AgentCore Runtime</strong></a> — Deploy agents securely into your VPC with built-in scaling and observability.</li> 
 <li><a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html" rel="noopener noreferrer" target="_blank"><strong>AgentCore Memory</strong></a> — Retain findings across sessions so agents learn from past investigations.</li> 
</ul> 
<p>The following diagram illustrates the end-to-end workflow from operator prompt to diagnostic report.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/23/DBBLOG-5235-1.png" /></p> 
<p>Flow: </p> 
<ol> 
 <li>DBA sends prompt — “Check for deadlocks in the last 24 hours and send a diagnostic report via SNS”.</li> 
 <li>Agent sends to LLM for reasoning — The model reads the prompt and the system prompt to understand the task.</li> 
 <li>LLM selects tools — Based on the docstrings, it decides to call get_deadlock_graphs first.</li> 
 <li>Agent executes tool — Queries system_health XE files on Amazon RDS for SQL Server for xml_deadlock_report events </li> 
 <li>Tool returns results — Deadlock graphs with session IDs, SQL statements, lock types, and objects.</li> 
 <li>Agent sends results to LLM for analysis — The model interprets the raw deadlock XML and identifies root cause.</li> 
 <li>LLM generates response — Produces a diagnosis with severity, root cause, affected queries, and remediation steps. Decides to call send_diagnostic_report to deliver findings via SNS.</li> 
 <li>Agent returns response to DBA — The DBA receives both the inline response and an email with the full diagnostic report.</li> 
</ol> 
<h2>Deploying to AgentCore Runtime</h2> 
<p>The agent is defined and the tools are ready. The complete source code, including both agent implementations, IAM policies, and deployment instructions, is available on GitHub. Refer to <a href="https://github.com/aws-samples/sample-agentcore-sqlserver-dbops-agent" rel="noopener noreferrer" target="_blank">An AI Agent for Deadlock Analysis on Amazon Amazon RDS for SQL Server</a>.</p> 
<p>Let’s walk through getting it running on AgentCore Runtime.</p> 
<h3>Prerequisites</h3> 
<p>The following resources are required for this walkthrough. You can use existing resources or create new ones. Each resource maps to an environment variable used during deployment.</p> 
<p><strong>Amazon RDS for SQL Server</strong></p> 
<ul> 
 <li>DB_INSTANCE_ID — An RDS for SQL Server instance (Standard or Enterprise Edition)</li> 
 <li>Trace flags 1204 and 1222 enabled via a custom DB parameter group. See <a href="https://aws.amazon.com/blogs/database/monitor-deadlocks-in-amazon-rds-for-sql-server-and-set-notifications-using-amazon-cloudwatch/" rel="noopener noreferrer" target="_blank">Monitor deadlocks in Amazon RDS for SQL Server</a>.</li> 
 <li>A custom extended event session for blocked_process_report for historical blocking capture. See <a href="https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.

ExtendedEvents.html" rel="noopener noreferrer" target="_blank">Using extended events with Amazon RDS for SQL Server</a>.</li> 
 <li>DB_SECRET_ID — Database credentials stored in <a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> with a dedicated, least-privilege database login.</li> 
 <li>SNS_TOPIC_NAME — An <a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> topic with an active subscription</li> 
</ul> 
<p><strong>Amazon Bedrock AgentCore</strong></p> 
<ul> 
 <li>AWS_REGION — <a href="https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html" rel="noopener noreferrer" target="_blank">Access to Amazon Bedrock foundation models</a> enabled in your region</li> 
 <li>AGENTCORE_ROLE_ARN — An IAM execution role with permissions for Bedrock, Secrets Manager, SNS, and CloudWatch Logs. See <a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-permissions.html" rel="noopener noreferrer" target="_blank">AgentCore Runtime permissions</a>.</li> 
 <li>SUBNET1, SUBNET2, SECURITY_GROUP_ID — A VPC with private subnets and a security group allowing outbound traffic to RDS on port 1433. See <a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-vpc.html" rel="noopener noreferrer" target="_blank">AgentCore VPC configuration</a>.</li> 
</ul> 
<p><strong>Development environment</strong></p> 
<ul> 
 <li>Python 3.10 or newer</li> 
 <li>AWS Command Line Interface (AWS CLI) configured with appropriate permissions</li> 
</ul> 
<h3>Step 1: Clone and install</h3> 
<div class="hide-language"> 
 <pre><code class="lang-code">git clone https://github.com/aws-samples/sample-agentcore-sqlserver-dbops-agent.git

cd sample-agentcore-sqlserver-dbops-agent

uv venv &amp;&amp; source .venv/bin/activate
uv pip install -r requirements.txt
uv pip install bedrock-agentcore-starter-toolkit</code></pre> 
</div> 
<h3>Step 2: Set environment variables</h3> 
<p>Set the following environment variables for your deployment. These reference the resources created in the prerequisites:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">export AWS_REGION=&lt;your-region&gt;<br />export DB_INSTANCE_ID=&lt;your-rds-instance-id&gt;       # RDS SQL Server instance<br />export DB_SECRET_ID=&lt;your-secrets-id&gt;            # Secrets Manager credential<br />export SNS_TOPIC_NAME=&lt;your-sns-topic-name&gt;         # SNS topic for reports<br />export AGENTCORE_ROLE_ARN=&lt;your-execution-role-arn&gt;    # IAM execution role<br />export SECURITY_GROUP_ID=&lt;your-security-group-id&gt;      # VPC security group<br />export SUBNET1=&lt;your-first-subnet-id&gt;                  # VPC private subnet<br />export SUBNET2=&lt;your-second-subnet-id&gt;                 # VPC private subnet</code></pre> 
</div> 
<h3>Step 3: Configure and deploy</h3> 
<p>Use the AgentCore CLI to configure and deploy the agent into your VPC:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">agentcore configure \
--name my_first_agentcore_runtime \
--entrypoint agent.py \
--execution-role $AGENTCORE_ROLE_ARN \
--deployment-type direct_code_deploy \
--vpc \
--subnets $SUBNET1,$SUBNET2 \
--security-groups $SECURITY_GROUP_ID</code></pre> 
</div> 
<div class="hide-language"> 
 <pre><code class="lang-code">agentcore deploy \
--env AWS_REGION=$AWS_REGION \
--env DB_SECRET_ID=$DB_SECRET_ID \
--env SNS_TOPIC_NAME=$SNS_TOPIC_NAME \
--env AGENT_OBSERVABILITY_ENABLED=true</code></pre> 
</div> 
<div class="hide-language"> 
 <pre><code class="lang-code">agentcore status</code></pre> 
</div> 
<h3>Step 4: Test your deployed agent</h3> 
<p>First, simulate a deadlock by creating two tables (##Employees and ##Suppliers) and running two concurrent transactions that update them in opposite order. Session 1 updates ##Employees then ##Suppliers, while Session 2 updates ##Suppliers then ##Employees. This creates a lock ordering conflict that SQL Server resolves by terminating one transaction. Follow the steps in <a href="https://github.com/aws-samples/sample-agentcore-sqlserver-dbops-agent?tab=readme-ov-file#testing-with-a-simulated-deadlock" rel="noopener noreferrer" target="_blank">Testing with a simulated deadlock</a></p> 
<p>Then invoke the agent:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">agentcore invoke '{"prompt": "Check for deadlocks in the last 24 hours, provide RCA and recommendations, and send a diagnostic report via SNS"}'</code></pre> 
</div> 
<p>The agent reads the deadlock graph from system_health, identifies both sessions and their SQL statements, correlates with active blocking chains, and sends a diagnostic report via SNS with root cause analysis and remediation steps.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">demo-server $ 

demo-server $ agentcore invoke '{"prompt": "Check for deadlocks in the last 24 hours, provide RCA and recommendations, and send a diagnostic report via SNS"}'

╭───────────────────────────────────────────────────────────────────── my_first_agentcore_runtime ─────────────────────────────────────────────────────────────────────╮

│ Session: bb18d342-0987-4854-bc8f-bb5c0edff095                                                                                                                                                                                                                                                                                                      │

│ Request ID: 554ff565-e99e-430c-b79c-a6e480d14a06                                                                                                                                                                                                                                                                                                │

│ ARN: arn:aws:bedrock-agentcore:us-west-2:111122223333:runtime/my_first_agentcore_runtime-MnC7NP39B6                                                                                                                                                                      │

│ Logs: aws logs tail /aws/bedrock-agentcore/runtimes/my_first_agentcore_runtime-MnC7NP39B6-DEFAULT --log-stream-name-prefix "2026/03/25/[runtime-logs" --follow                                              │

│       aws logs tail /aws/bedrock-agentcore/runtimes/my_first_agentcore_runtime-MnC7NP39B6-DEFAULT --log-stream-name-prefix "2026/03/25/[runtime-logs" --since 1h                                               │

│ GenAI Dashboard: https://console.aws.amazon.com/cloudwatch/home?region=us-west-2#gen-ai-observability/agent-core                                                                                                                                                                                                                                                                                                                                                                                                         │

╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

Response:

## Comprehensive Analysis Complete

I've conducted a thorough deadlock investigation and delivered a detailed diagnostic report. Here's my assessment:

### **Key Findings:**

- **1 deadlock** detected at 07:16:01 on March 25, 2026

- **Severity: WARNING** - Low frequency but requires proactive intervention

- **Root Cause:** Lock ordering inconsistency between Suppliers and Employees table updates

### **Root Cause Analysis:**

The deadlock occurred due to concurrent transactions updating related records (both with ID = 1) in different order, creating a circular wait condition. This suggests 

either:

- Foreign key relationships between tables

- Application logic that doesn't follow consistent lock ordering

- Potential business process coupling between supplier and employee data

### **Critical Recommendations Delivered:**

1. **Immediate:** Implement deadlock retry logic and consistent lock ordering

2. **Short-term:** Enable READ_COMMITTED_SNAPSHOT and enhance monitoring  

3. **Long-term:** Architectural review and performance optimization

### **Actions Completed:**

✅ Comprehensive deadlock analysis  

✅ Detailed root cause analysis  

✅ Prioritized recommendations with timelines  

✅ Prevention strategy outlined  

✅ Monitoring plan established  

✅ Diagnostic report sent via SNS to DBA team  

The report includes specific technical details, business impact assessment, and a structured action plan with priorities and timelines to prevent future occurrences.

demo-server $</code></pre> 
</div> 
<h2>Best practices</h2> 
<p><strong>Cross-session memory.</strong> Without memory, every invocation starts fresh. With <a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html" rel="noopener noreferrer" target="_blank">AgentCore Memory</a>, findings persist across sessions through two complementary systems: short-term memory (STM) stores raw conversation events for session continuity, while long-term memory (LTM) extracts durable insights that carry across invocations.</p> 
<p>LTM supports multiple extraction strategies. The <strong>semantic</strong> strategy captures facts that remain relevant over time: “Recurring deadlocks between Employees and Suppliers updates because of opposite lock ordering.” The <strong>summary</strong> strategy condenses investigation sessions: “Investigation found 6 deadlocks involving ##Employees and ##Suppliers tables and a 14-session blocking chain caused by an index rebuild.” Transient metrics like current CPU utilization are correctly skipped; only durable patterns are retained.</p> 
<p>Create a memory resource with both strategies:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">agentcore memory create dbops_shared_memory \<br />  --strategies '[{"semanticMemoryStrategy": {"name": "dbops_facts"}}, {"summaryMemoryStrategy": {"name": "dbops_summaries"}}]' \<br />  --event-expiry-days 30 \<br />  --region $AWS_REGION \<br />  --wait
AGENTCORE_MEMORY_ID=$(agentcore memory list --region $AWS_REGION 2&gt;/dev/null | grep dbops_shared_memory | awk '{print $4}')</code></pre> 
</div> 
<p>Integrating memory requires a code change to agent.py. Add the <a href="https://strandsagents.com/docs/community/session-managers/agentcore-memory/" rel="noopener noreferrer" target="_blank">AgentCoreMemorySessionManager</a> as the session manager. See <a href="https://github.com/aws-samples/sample-agentcore-sqlserver-dbops-agent?tab=readme-ov-file#adding-memory-optional" rel="noopener noreferrer" target="_blank">agent_with_memory.py</a> for the complete example.</p> 
<ol> 
 <li>On Monday, the agent finds 6 deadlocks involving updates to ##Employees and ##Suppliers tables in opposite lock order, and sends a diagnostic report via SNS.</li> 
 <li>On Tuesday, the DBA deploys a lock ordering fix based on the agent’s recommendation.</li> 
 <li>On Wednesday, the alarm fires again. The agent recalls Monday’s findings from memory, runs a fresh investigation, and compares: “The Employees/Suppliers deadlock pattern has stopped. New deadlock detected between different tables.”</li> 
</ol> 
<p><strong>Observability</strong></p> 
<p>To enable observability:</p> 
<ul> 
 <li>Add <a href="https://aws.amazon.com/otel/" rel="noopener noreferrer" target="_blank">aws-opentelemetry-distro</a> to your requirements.txt file.</li> 
 <li>Set the environment variable AGENT_OBSERVABILITY_ENABLED=true during deployment.</li> 
 <li>AgentCore automatically instruments traces, token usage, and request duration. </li> 
 <li>No code changes required. </li> 
 <li>View metrics in the <a href="https://console.aws.amazon.com/cloudwatch/home#gen-ai-observability/agent-core" rel="noopener noreferrer" target="_blank">CloudWatch GenAI Observability dashboard</a>.</li> 
</ul> 
<p><strong>Scaling with additional tools.</strong> </p> 
<p>The same @tool pattern applies to other diagnostic scripts:</p> 
<ul> 
 <li>Slow query analysis via Query Store</li> 
 <li>Index recommendations from sys.dm_db_missing_index_details</li> 
 <li>TempDB troubleshooting</li> 
</ul> 
<p>Strands also supports <a href="https://strandsagents.com/docs/user-guide/concepts/tools/mcp-tools/" rel="noopener noreferrer" target="_blank">Model Context Protocol (MCP)</a>, allowing agents to connect to external tool servers and extend their capabilities without modifying agent code.</p> 
<h2>Key takeaways</h2> 
<ul> 
 <li>The system_health extended event session captures deadlock graphs by default. The agent reads them and delivers root cause analysis.</li> 
 <li>Blocking chain analysis uses DMVs to identify head blockers and their SQL. Historical blocker identity requires blocked process threshold and a custom extended event session.</li> 
 <li>The @tool decorator connects your existing T-SQL queries to the LLM. The docstring determines when the agent calls each tool.</li> 
 <li>Correlation across data sources — connecting deadlocks to transaction errors and blocking to timeouts — is what distinguishes an agent from individual scripts.</li> 
 <li>Strands Agents provides the tool framework, AgentCore Runtime provides managed deployment, and together they turn your existing T-SQL expertise into production-ready agents.</li> 
 <li>Design tools that are focused (do one thing well), efficient (return only necessary data), reliable (handle errors gracefully), fast (execute quickly), and clear (well-documented). Good tool design is what makes an agent effective.</li> 
</ul> 
<h2>Clean up</h2> 
<p>Complete the following steps to clean up your resources:</p> 
<ol> 
 <li>On the Amazon RDS console, choose Databases in the navigation pane.</li> 
 <li>Select the DB instance to delete.</li> 
 <li>On the Actions menu, choose Delete.</li> 
 <li>Enter delete me to confirm deletion, then choose Delete.</li> 
 <li>When prompted to create a final snapshot and retain automated backup, choose the option appropriate for your needs.</li> 
</ol> 
<p>To delete the <a href="https://aws.github.io/bedrock-agentcore-starter-toolkit/api-reference/cli.html" rel="noopener noreferrer" target="_blank">AgentCore Runtime</a> resource:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">agentcore destroy --agent my_first_agentcore_runtime</code></pre> 
</div> 
<p>To delete the <a href="https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-get-started.html#memory-cleanup" rel="noopener noreferrer" target="_blank">AgentCore Memory</a> resource:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">agentcore memory delete $AGENTCORE_MEMORY_ID --region $AWS_REGION</code></pre> 
</div> 
<h2>Conclusion</h2> 
<p>In this post, we showed how to turn existing T-SQL diagnostic scripts into an AI-powered database operations agent using Strands Agents and Amazon Bedrock AgentCore. The agent investigates deadlocks and blocking on Amazon RDS for SQL Server, correlates findings across data sources, and delivers actionable recommendations, all without manual intervention.</p> 
<p>To get started, clone the sample repository and deploy the agent against your own RDS for SQL Server instance. Try extending it with additional tools for your most common diagnostic workflows, such as Query Store analysis, index recommendations, or TempDB troubleshooting. We’d love to hear what you build. Share your feedback and questions in the comments section.</p> 
<hr /> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Sudhir Amin" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2025/11/21/DBBLOG-5231-5.png" width="120" />
  </div> 
  <h3 class="lb-h4">Sudhir Amin</h3> 
  <p><a href="https://www.linkedin.com/in/sudhir-amin-703a2244/" rel="noopener" target="_blank">Sudhir</a> is a Database Specialist Solutions Architect at Amazon Web Services. In his role based out of New York, he provides architectural guidance and technical assistance to enterprise customers across different industry verticals, accelerating their cloud adoption. He is a big fan of snooker, combat sports such as boxing and UFC, and loves traveling to countries with rich wildlife reserves where he gets to see world’s most majestic animals up close.</p>
 </div> 
</footer>
