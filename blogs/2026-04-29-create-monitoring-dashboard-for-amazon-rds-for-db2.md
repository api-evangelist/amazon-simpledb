---
title: "Create monitoring dashboard for Amazon RDS for Db2"
url: "https://aws.amazon.com/blogs/database/create-monitoring-dashboard-for-amazon-rds-for-db2/"
date: "Wed, 29 Apr 2026 15:24:14 +0000"
author: "Vikram Khatri"
feed_url: "https://aws.amazon.com/blogs/database/feed/"
---
<p><a href="https://aws.amazon.com/rds/db2/" rel="noopener" target="_blank">Amazon Relational Database Service (Amazon RDS) for Db2</a> is a fully managed relational database service that simplifies setting up, operating, and scaling IBM Db2 databases in the cloud. It automates time-consuming administration tasks such as hardware provisioning, database setup, patching, and backups — freeing you to focus on building and optimizing your applications. However, to maintain optimal performance and verify application health, you need to continuously monitor your database metrics.</p> 
<p>In this post, we walk you through deploying an automated Amazon CloudWatch monitoring dashboard for Amazon RDS for Db2. This solution works for both internet-connected (online) and private subnet (air-gapped) environments, requiring no manual console steps. A single shell script orchestrates the entire deployment. It provisions an <a href="https://aws.amazon.com/lambda/" rel="noopener" target="_blank">AWS Lambda</a> function, <a href="https://aws.amazon.com/eventbridge/" rel="noopener" target="_blank">Amazon EventBridge</a> schedules, <a href="https://aws.amazon.com/vpc/" rel="noopener" target="_blank">Amazon VPC</a> endpoints, and a comprehensive <a href="https://aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">Amazon CloudWatch</a> dashboard that provides real-time visibility into your database performance.</p> 
<h2>Solution overview</h2> 
<p>The solution uses a shell script (<code>db2monitor.sh</code>) that orchestrates an AWS CloudFormation-based deployment. It automates the following:</p> 
<ul> 
 <li><strong>Creates a Lambda function</strong> (Python 3.12) with an <code>ibm_db</code> layer to connect securely to your RDS for Db2 instance</li> 
 <li><strong>Stores database credentials </strong>in AWS Secrets Manager for secure access</li> 
 <li><strong>Schedules metric collection</strong> via Amazon EventBridge Scheduler at regular intervals</li> 
 <li><strong>Publishes performance metrics </strong>to Amazon CloudWatch under the <code>RDS-DB2-MON</code> namespace</li> 
 <li><strong>Builds a CloudWatch dashboard </strong>scoped to your DB instance, database name, and deployment tag for easy identification</li> 
</ul> 
<p>For air-gapped (private subnet) environments, a separate script (<code>db2mon-airgap.sh</code>) pre-stages all artifacts in a private S3 bucket, so that no internet access is needed at deployment time.</p> 
<p><strong>Architecture diagram for Amazon RDS for Db2 monitoring solution</strong></p> 
<p>The architecture illustrates how Amazon EventBridge Scheduler triggers a Lambda function at regular intervals to collect performance metrics from your Amazon RDS for Db2 instance. The Lambda function securely retrieves database credentials from AWS Secrets Manager and connects to the database through the VPC. It then publishes custom metrics to Amazon CloudWatch, which powers a comprehensive monitoring dashboard. For air-gapped (private subnet without Network Address Translation (NAT) deployments), all AWS service interactions occur through VPC interface endpoints, which provides complete isolation from the public internet.</p> 
<p><strong></strong></p>
<strong> <p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-1.jpg" /></p> </strong>
<p><strong></strong></p> 
<h2>Prerequisites</h2> 
<h3>Online deployment</h3> 
<p>For deployments from AWS CloudShell or an Amazon Elastic Compute Cloud (Amazon EC2) instance with internet access, you need:</p> 
<ul> 
 <li><strong>AWS Command Line Interface (AWS CLI) v2</strong> installed and configured</li> 
 <li><strong>jq </strong>(JSON processor) installed</li> 
 <li><strong>AWS Identity and Access Management (AWS IAM) permissions</strong> to create AWS Lambda, AWS CloudFormation, Amazon EventBridge, Amazon Secrets Manager, AWS Systems Manager, Amazon Simple Storage Service (Amazon S3), Amazon CloudWatch, and Amazon VPC resources</li> 
 <li>Run <code>./db2monitor.sh --check-permissions --region &lt;region&gt;</code> to validate your permissions before deployment</li> 
</ul> 
<h3>Air-gapped deployment</h3> 
<p>For deployments in private subnets without internet access, you need:</p> 
<ul> 
 <li>A machine with internet access to download artifacts</li> 
 <li>A machine with AWS CLI access to upload artifacts to S3</li> 
 <li>A private-subnet EC2 instance to run the deployment</li> 
 <li>The following VPC endpoints must exist <strong>before </strong>deployment:</li> 
</ul> 
<table border="1px" cellpadding="10px"> 
 <tbody>
  <tr> 
   <td><strong>Type </strong></td> 
   <td><strong>Service</strong></td> 
  </tr> 
  <tr> 
   <td>Gateway</td> 
   <td>s3</td> 
  </tr> 
  <tr> 
   <td>Interface</td> 
   <td>ssm, ssmmessages, ec2messages, ec2</td> 
  </tr> 
 </tbody>
</table> 
<p>Note: </p> 
<ul> 
 <li>Script <code>db2monitor.sh</code> automatically creates the remaining required endpoints (sts, secretsmanager, monitoring, logs, lambda, rds, sns, sqs, scheduler, cloudformation) during deployment.</li> 
 <li>VPC DNS settings: <code>enableDnsSupport=true</code>, <code>enableDnsHostnames= true</code></li> 
</ul> 
<h2>Deploy Lambda to build Amazon RDS for Db2 dashboard</h2> 
<p><strong>Online deployment</strong></p> 
<p><strong>Step 1: Download the script</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-code">curl -sL https://bit.ly/db2monitor</code></pre> 
</div> 
<p>Note: The above URL points to <code>https://aws-blogs-artifacts-public.s3.us-east-1.amazonaws.com/artifacts/DBBLOG-3742/db2mon-unified.sh</code></p> 
<p><strong>Step 2: (Optional) Pre-configure environment variables</strong></p> 
<p>If you prefer to set variables once rather than passing them inline:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vi db2monitor.env # Edit the file with your values

source db2monitor.env</code></pre> 
</div> 
<p><strong>Step 3: Deploy</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./db2monitor.sh --region &lt;region&gt;</code></pre> 
</div> 
<p>Alternatively, pass all options inline for a single-command deployment:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">REGION=&lt;region&gt; DB_INSTANCE_ID=&lt;mydb2id&gt; \
DBNAME=&lt;dbname&gt; TAG=&lt;tagName&gt; \

./db2monitor.sh</code></pre> 
</div> 
<p><strong>Air-gapped deployment</strong></p> 
<p><strong>Step 1: Download artifacts (from an internet-connected machine)</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-code">curl -sL https://bit.ly/db2monitor | bash

./db2mon-airgap.sh --mode download --region &lt;region&gt;</code></pre> 
</div> 
<p>This saves all artifacts to <code>./db2mon-artifacts/</code>.</p> 
<p><strong>Step 2: Upload artifacts to your private S3 bucket</strong></p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./db2mon-airgap.sh --mode upload --region &lt;region&gt;</code></pre> 
</div> 
<p>Or combine download and upload in one step if the machine has both internet and AWS access:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./db2mon-airgap.sh --mode both --region &lt;region&gt;</code></pre> 
</div> 
<p>This creates the bucket <code>s3://lambda-functions-&lt;account&gt;-&lt;region&gt;/.</code></p> 
<p><strong>Step 3: Deploy from your private-subnet EC2 instance</strong></p> 
<p><em># Pull the script from your private bucket (via S3 Gateway endpoint)</em></p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws s3 cp s3://lambda-functions-&lt;account&gt;-&lt;region&gt;/db2monitor.sh . \

chmod +x db2monitor.sh

<em># Deploy</em>

BUCKET=lambda-functions-&lt;account&gt;-&lt;region&gt; \
REGION=&lt;region&gt; \
DB_INSTANCE_ID=&lt;mydb2id&gt; DBNAME=&lt;dbName&gt; TAG=&lt;tagName&gt; \

./db2monitor.sh</code></pre> 
</div> 
<p><strong>Post-deployment modules</strong></p> 
<p>After the initial install, use <code>--module</code> to manage the deployment:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./db2monitor.sh --module start    # Enable monitoring schedule
./db2monitor.sh --module stop     # Disable monitoring schedule
./db2monitor.sh --module refresh  # Reload secret values</code></pre> 
</div> 
<p><strong>Resource naming</strong></p> 
<p>The deployment creates resources with predictable names scoped to your instance, database, and tag:</p> 
<table border="1px" cellpadding="10px"> 
 <tbody>
  <tr> 
   <td><strong>Resource</strong></td> 
   <td><strong>Name</strong></td> 
  </tr> 
  <tr> 
   <td>CloudFormation stack</td> 
   <td>DB2-Dashboard-&lt;DB_INSTANCE_ID&gt;-&lt;DBNAME&gt;-&lt;TAG&gt;</td> 
  </tr> 
  <tr> 
   <td>Lambda function</td> 
   <td>DB2Mon-Lambda-Function-&lt;region&gt;</td> 
  </tr> 
  <tr> 
   <td>Secrets Manager secret</td> 
   <td>SM-&lt;DB_INSTANCE_ID&gt;-&lt;DBNAME&gt;-&lt;TAG&gt;</td> 
  </tr> 
  <tr> 
   <td>SSM registry</td> 
   <td>/db2mon/instances</td> 
  </tr> 
  <tr> 
   <td>EventBridge schedule</td> 
   <td>db2mon-cw-&lt;DB_INSTANCE_ID&gt;-&lt;DBNAME&gt;-&lt;TAG&gt;</td> 
  </tr> 
  <tr> 
   <td>S3 bucket</td> 
   <td>lambda-functions-&lt;account&gt;-&lt;region&gt;</td> 
  </tr> 
 </tbody>
</table> 
<h3>Diagnostics script</h3> 
<p>Run the diagnostics script <strong>before</strong> deploying to catch common issues:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./db2mon-diag.sh --region &lt;region&gt;</code></pre> 
</div> 
<p>This script validates:</p> 
<ul> 
 <li>AWS credentials and AWS Region configuration</li> 
 <li>VPC endpoints (all required services)</li> 
 <li>Security group rules (inbound DB2 port, outbound 443)</li> 
 <li>Network connectivity to the RDS endpoint</li> 
 <li>IAM permissions</li> 
 <li>S3 bucket and artifact presence</li> 
</ul> 
<p><strong>Common issues:</strong></p> 
<table border="1px" cellpadding="10px"> 
 <tbody>
  <tr> 
   <td><strong>Symptom</strong></td> 
   <td><strong>Root Cause</strong></td> 
   <td><strong>Solution</strong></td> 
  </tr> 
  <tr> 
   <td>Lambda cannot reach Amazon RDS</td> 
   <td>Security group misconfiguration</td> 
   <td>Verify that the security group allows inbound traffic on the Db2 port (default 50000) and outbound traffic on port 443 for 0.0.0.0/0.</td> 
  </tr> 
  <tr> 
   <td>Air-gap deploy fails immediately</td> 
   <td>Missing VPC endpoints</td> 
   <td>Amazon S3 gateway endpoint and Amazon EC2 interface endpoint must exist before running db2monitor.sh</td> 
  </tr> 
  <tr> 
   <td>Metrics not appearing in Amazon CloudWatch</td> 
   <td>EventBridge schedule disabled or Lambda errors</td> 
   <td>Is the EventBridge schedule enabled? Run <code>--module</code> start. Check Lambda logs in CloudWatch Logs.</td> 
  </tr> 
  <tr> 
   <td>Permission errors during deployment</td> 
   <td>Insufficient IAM permissions</td> 
   <td>Run <code>--check-permissions</code> and attach missing IAM actions.</td> 
  </tr> 
  <tr> 
   <td>Connection timeout to Amazon RDS</td> 
   <td>Network routing or DNS issues</td> 
   <td>Verify the VPC.</td> 
  </tr> 
 </tbody>
</table> 
<h2>Cost considerations</h2> 
<p>This monitoring solution incurs minimal operational costs. The primary cost components include Lambda invocations for metric collection, CloudWatch custom metrics and dashboard usage, Secrets Manager for credential storage, and Amazon CloudWatch Logs retention. Additional costs apply for VPC interface endpoints required for private subnet connectivity for Lambda function invocations.</p> 
<p><strong>Key cost factors:</strong></p> 
<ul> 
 <li>Lambda execution frequency (default: 1-minute intervals)</li> 
 <li>Number of custom metrics published to CloudWatch</li> 
 <li>VPC interface endpoints for Lambda function</li> 
 <li>CloudWatch Logs retention period</li> 
</ul> 
<p><strong>Cost optimization:</strong> Ahare VPC endpoints across multiple solutions, and configure appropriate log retention policies.</p> 
<h2>Cleanup </h2> 
<div class="hide-language"> 
 <pre><code class="lang-code">./db2mon-cleanup.sh --region &lt;region&gt;</code></pre> 
</div> 
<h2>Try this solution</h2> 
<p>The source code of the scripts is available in the <a href="https://github.com/aws-samples/sample-rds-db2-tools/tree/main/tools/db2monitor" rel="noopener" target="_blank">https://github.com/aws-samples/sample-rds-db2-tools/tree/main/tools/db2monitor</a>. <a href="https://github.com/aws-samples/sample-rds-db2-tools/issues" rel="noopener" target="_blank">Open an issue</a> to submit your enhancements request or submit a <a href="https://github.com/aws-samples/sample-rds-db2-tools/pulls" rel="noopener" target="_blank">pull request</a> with your suggested changes.</p> 
<h2>Db2 Monitor Dashboard</h2> 
<p>Upon deploying the dashboard and activating the event bridge to commence monitoring your Amazon RDS for Db2 database, you will encounter a multitude of dashboard widgets. For illustrative purposes, several screenshots of these widgets are presented below, providing a visual representation of their appearance and functionality.</p> 
<p>The provided screenshot illustrates the number of CPUs configured and available online for the RDS for Db2 system. It also displays the variation in CPU speed at the server and CPU load over a 1-minute, 10-minute, and 15-minute period. Additionally, the database memory utilization with High Water Marks (HWM) is depicted. </p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-2.png" /></p> 
<p>The provided screen snapshot illustrates the activities monitored over the specified time range in the CloudWatch console. For instance, selecting a 24-hour range will display the total SQL activities and rows processed by the RDS for Db2 database. These key performance indicators (KPIs) serve as crucial indicators of the workload type of the database being monitored. </p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-3.png" /></p> 
<p>The provided screen snapshot illustrates the database time breakdown, encompassing wait time across the specified range. Additionally, the distribution of wait time over the range is depicted. This data serves as a valuable tool for analyzing the time allocated to various database activities, enabling further investigation.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-4.png" /></p> 
<p>The provided screen capture illustrates the cost-intensive SQL statements categorized for targeted enhancement.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-5.png" /></p> 
<p>The provided screenshot illustrates the Db2 buffer pool activities for data and index, including the log write time information.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-6.png" /></p> 
<p>Note: The dashboard contains numerous widgets, with only a select few being displayed above. </p> 
<p>The CloudWatch dashboard provides a feature called “Play” that allows users to activate it to view widgets over a specified time. An illustrative example is provided below.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-7.gif" /></p> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to deploy a fully automated Amazon CloudWatch monitoring dashboard for Amazon RDS for Db2 using a single shell script. This solution works in both internet-connected and air-gapped environments. It eliminates manual console configuration and provides comprehensive visibility into your database performance. Whether you’re running in a standard VPC or a highly secure private subnet, the deployment process remains the same. When you no longer need the monitoring infrastructure, you can run a single cleanup script to remove all resources.</p> 
<hr /> 
<h2>Acknowledgments</h2> 
<p>Thanks to <a href="https://www.linkedin.com/in/rajibsarkaribm/" rel="noopener" target="_blank">Rajib Sarkar</a> and <a href="https://www.linkedin.com/in/uhussain/" rel="noopener" target="_blank">Umair Hussain</a> for carefully reviewing this blog post.</p> 
<h2>About the authors</h2> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Vikram S Khatri" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-8.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Vikram S Khatri</h3> 
  <p><a href="https://www.linkedin.com/in/viz7/" rel="noopener" target="_blank">Vikram</a> is a Senior Engineer for Amazon RDS for Db2. He holds multiple roles, including Product Management, Experienced Architect, Leadership, and AI Expert User. With over 20 years of experience, Vikram is passionate about developing innovative products from scratch. In his personal time, he listens to podcasts <a href="https://www.youtube.com/@allin" rel="noopener" target="_blank">All-in</a>, <a href="https://www.youtube.com/@DwarkeshPatel" rel="noopener" target="_blank">Dwarkesh</a>, <a href="https://www.youtube.com/@LennysPodcast" rel="noopener" target="_blank">Lenny</a>, <a href="https://www.youtube.com/lexfridman" rel="noopener" target="_blank">Lex Friedman</a>, <a href="https://www.youtube.com/@peterdiamandis" rel="noopener" target="_blank">Moonshots</a>, <a href="https://www.youtube.com/andrejkarpathy" rel="noopener" target="_blank">Karpathy</a> and others. </p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Sumit Kumar" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-9.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">Sumit Kumar</h3> 
  <p><a href="https://www.linkedin.com/in/sumitkumarsangerpal/" rel="noopener" target="_blank">Sumit</a> is a Senior Solutions Architect at AWS, and enjoys solving complex problems. He has been helping customers across various industries to build and design their workloads on the AWS Cloud. He enjoys cooking, playing chess, and spending time with his family.</p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="YunCheol Ha" class="aligncenter size-full wp-image-29797" height="160" src="https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2026/04/21/DBBLOG-5467-10.jpeg" width="120" />
  </div> 
  <h3 class="lb-h4">YunCheol Ha</h3> 
  <p><a href="https://www.linkedin.com/in/yuncheol-ha-02107622/" rel="noopener" target="_blank">YunCheol</a> is a Senior Specialist Solutions Architect at AWS. He works with customers across the Asia-Pacific region to design, migrate, and optimize their database workloads on AWS. Passionate about database performance optimization, he is dedicated to delivering high-performance and scalable solutions. In his free time, he loves swimming in pools and the ocean.</p>
 </div> 
</footer>
