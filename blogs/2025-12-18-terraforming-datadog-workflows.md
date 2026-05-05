---
title: "Terraforming Datadog Workflows"
url: "https://runa.io/blog/terraforming-datadog-workflows/"
date: "Thu, 18 Dec 2025 23:10:01 +0000"
author: "Olakitan Aladesuyi"
feed_url: "https://runa.io/feed"
---
<p>Terraform recently added the datadog_workflow_automation resource to its <a href="https://registry.terraform.io/providers/DataDog/datadog/3.58.0/docs">Datadog registry</a>. This new resource enables teams to provision and manage Datadog workflows using Terraform, giving teams the ease and flexibility of infrastructure-as-code for observability automation.</p>
<p><span id="more-27069"></span></p>
<p style="font-weight: normal;">At Runa, we have found workflows to be especially useful in operations management, alert optimization, and incident flow automation. However, managing these workflows solely via the Datadog UI quickly becomes difficult. Changes are hard to track, reviews are not centralized, and small edits can easily get lost. Combined with the challenge of maintaining workflows that serve multiple teams with different priorities, these gaps become painful and error-prone.</p>
<p style="font-weight: normal;">This why I have spent the past few months migrating two of our existing operations management workflows to Terraform. In this post, I will take you through the key technical challenges, the solutions, and the pitfalls to avoid when adopting Terraform for Datadog workflows.</p>
<p style="font-weight: normal;">&nbsp;</p>
<h2><strong>Key Technical Challenges and Solutions</strong></h2>
<h3><strong>Complex JSON Structure Management</strong></h3>
<p>The most significant challenge was managing the complex JSON structure required for the <code class="not-prose"><span style="color: #ff0201;">spec_json</span></code> parameter. Datadog workflows have intricate step definitions with nested parameters, outbound edges, and display configurations.</p>
<p><strong>Solution</strong>: Export the workflow from the Datadog console as Terraform code.</p>
<p>The Datadog console provides an &#8220;Export&#8221; option for existing workflows that outputs a complete Terraform representation of the workflow. This gives you the exact JSON format that Datadog expects, minimizing trial-and-error. From there, you can refactor the <span style="color: #ff0201;"><code class="not-prose">spec_json</code></span> using <span style="color: #ff0201;"><code class="not-prose">jsonencode()</code></span> and Terraform locals to make it more modular and maintainable.</p>
<p><strong>Steps to export</strong>:</p>
<ol>
<li>Go to your workflow in the Datadog console.</li>
<li>Click on the<span style="color: #ff0201;"> <code class="not-prose">Export</code></span> option in the menu bar.</li>
<li>Select the <code class="not-prose"><span style="color: #ff0201;">Terraform</span></code> option.</li>
<li>Copy the generated <code class="not-prose"><span style="color: #ff0201;">spec_json</span></code> content.</li>
</ol>
<p style="font-weight: normal;"><!-- notionvc: 1e510679-66c6-4502-9653-63fc24dde86b --></p>
<p><!-- notionvc: cf8a1c72-fd7c-4cb9-85c3-16af7aed4bdf --></p>
<h3><strong>Dynamic Step References and Dependencies</strong></h3>
<p>One of the trickiest aspects was handling dynamic references between steps that are conditionally included. For example, one workflow includes a call to another workflow only when a specific criterion is met, which means that the outbound edges (the “links” between steps) must reference steps that may or may not exist.</p>
<p><strong>Solution</strong>: Use Terraform locals to define conditional outbound edge maps. This approach allows you to model branching logic without duplicating JSON structures.</p>
<p><code class="not-prose">locals {</code><br /><code class="not-prose">&nbsp; check_monitor_outbound_edges = var.enable_notifications ? [</code><br /><code class="not-prose">&nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "nextStepName" : "SendSlackNotification",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "branchName" : "true"</code><br /><code class="not-prose">&nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; ] : []</code><br /><code class="not-prose">}</code></p>
<p><code class="not-prose">---</code></p>
<p><code class="not-prose">{</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "CheckMonitorStatus",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "actionId" : "com.datadoghq.core.if",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "parameters" : [ ... ],</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "outboundEdges" : local.check_monitor_outbound_edges,</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; }</code></p>
<h3><strong>AWS Services Integration and Connection Management</strong></h3>
<p>Both workflows integrate with multiple AWS services, which require authentication. On the Datadog console, AWS connection is easily managed by creating an appropriate connection for each environment. However, Terraform is environment agnostic, so you must supply the AWS connection for the correct environment being deployed.</p>
<p><strong>Solution</strong>: Maintain a mapping of pre-existing environment-specific AWS connections:</p>
<p><!-- notionvc: f08c64b9-4e6c-4629-9391-3fdb38013b70 --></p>
<p><code class="not-prose">locals {</code><br /><code class="not-prose">&nbsp; aws_connection_map = {</code><br /><code class="not-prose">&nbsp; &nbsp; "sandbox" &nbsp; &nbsp;= { connection_id : "12345", label : "SAMPLE_CONNECTION_1" }</code><br /><code class="not-prose">&nbsp; &nbsp; "dev" &nbsp; &nbsp;= { connection_id : "67890", label : "SAMPLE_CONNECTION_2" }</code><br /><code class="not-prose">&nbsp; }</code><br /><code class="not-prose">}</code></p>
<h3><strong>Dynamic workflow naming</strong></h3>
<p>Not a challenge, but worth mentioning to avoid all provisioned workflows having the same name.</p>
<p><strong>Solution</strong>: Use Terraform variables to parameterize workflow names<!-- notionvc: 813ee841-4a66-4efe-a10f-adfb3f7f84c7 --></p>
<p><code class="not-prose">resource "datadog_workflow_automation" "my_workflow" {</code><br /><code class="not-prose">&nbsp; name &nbsp; &nbsp; &nbsp; &nbsp;= "My-workflow-${var.team_name}-${var.env}"</code><br /><code class="not-prose">}</code></p>
<p>&nbsp;</p>
<h3><strong>JavaScript Code in Workflow Steps</strong></h3>
<p>Many workflow steps include JavaScript code for data transformation. Managing this code within Terraform strings can quickly become messy and hard to read.</p>
<p><strong>Best Practices</strong>:</p>
<ul>
<li>Write and test your code directly in the Datadog workflow <strong>JavaScript step editor</strong> or any IDE.</li>
<li>Maintain proper indentation and commenting.</li>
<li>Export the working code as JSON from Datadog or IDE once validated.</li>
</ul>
<p><!-- notionvc: ca7fe9ea-2bdd-497b-940e-8c0cd6a40d3d --></p>
<p><code class="not-prose">{</code><br /><code class="not-prose">&nbsp; "name" : "ParseOutput",</code><br /><code class="not-prose">&nbsp; "actionId" : "com.datadoghq.datatransformation.func",</code><br /><code class="not-prose">&nbsp; "parameters" : [</code><br /><code class="not-prose">&nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "name" : "script",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "value" : "// parse JSON string \nlet parsedFaults = $.Steps.DescribeOutput;\nlet trigger_monitor_url = $.Source.monitor.url;\n\n ... // rest of logic here"</code><br /><code class="not-prose">&nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; ]</code><br /><code class="not-prose">}</code></p>
<p>&nbsp;</p>
<h2><strong>Complete example: Putting It All Together</strong></h2>
<p>Here&#8217;s a sample workflow that demonstrates the patterns discussed above in a complete workflow.</p>
<p><!-- notionvc: 86c54799-7b87-4fb8-be72-1b5c28eb185a --></p>
<p><code class="not-prose">locals {</code><br /><code class="not-prose">&nbsp; # Create conditional notification steps</code><br /><code class="not-prose">&nbsp; notification_steps = var.enable_notifications ? [</code><br /><code class="not-prose">&nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "name" : "SendSlackNotification",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "actionId" : "com.datadoghq.slack.send_simple_message",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "parameters" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "teamId",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "value" : "MYSAMPLETEAM"</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; },</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "channel",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "value" : "#alerts"</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; },</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "text",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "value" : "Alert triggered: "</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; ],</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "display" : {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; "bounds" : {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "x" : 0,</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "y" : 432</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; ] : []</code></p>
<p><code class="not-prose">&nbsp; # Create conditional outbound edges&nbsp;</code><br /><code class="not-prose">&nbsp; # If notifications are enabled, route to notification step; otherwise, workflow ends</code><br /><code class="not-prose">&nbsp; check_monitor_outbound_edges = var.enable_notifications ? [</code><br /><code class="not-prose">&nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "nextStepName" : "SendSlackNotification",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "branchName" : "true"</code><br /><code class="not-prose">&nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; ] : []</code></p>
<p><code class="not-prose">&nbsp; sample_workflow_name = "Sample-workflow-${var.team_name}"</code><br /><code class="not-prose">&nbsp; aws_connection_map = {</code><br /><code class="not-prose">&nbsp; &nbsp; # AWS connection IDs for different environments. Already created on datadog console</code><br /><code class="not-prose">&nbsp; &nbsp; "sandbox" = { connection_id : "abcd", label : "MY_SAMPLE_AWS_CONNECTION_1" }</code><br /><code class="not-prose">&nbsp; &nbsp; "dev" &nbsp; &nbsp;= { connection_id : "abcdefg", label : "MY_SAMPLE_AWS_CONNECTION_2" }</code><br /><code class="not-prose">&nbsp; }</code><br /><code class="not-prose">&nbsp; aws_connection_id &nbsp; &nbsp;= &nbsp;local.aws_connection_map["sandbox"].connection_id</code><br /><code class="not-prose">&nbsp; aws_connection_label = local.aws_connection_map["sandbox"].label</code><br /><code class="not-prose">}</code></p>
<p><code class="not-prose">variable "enable_notifications" {</code><br /><code class="not-prose">&nbsp; type &nbsp; &nbsp; &nbsp; &nbsp;= bool</code><br /><code class="not-prose">&nbsp; description = "Enable Slack notifications in the workflow"</code><br /><code class="not-prose">&nbsp; default &nbsp; &nbsp; = false</code><br /><code class="not-prose">}</code></p>
<p><code class="not-prose">resource "datadog_workflow_automation" "sample_workflow" {</code><br /><code class="not-prose">&nbsp; name &nbsp; &nbsp; &nbsp; &nbsp;= local.sample_workflow_name</code><br /><code class="not-prose">&nbsp; description = "Sample workflow demonstrating Terraform patterns for Datadog workflow automation"</code><br /><code class="not-prose">&nbsp; tags &nbsp; &nbsp; &nbsp; &nbsp;= var.workflow_tags</code><br /><code class="not-prose">&nbsp; published &nbsp; = true</code></p>
<p><code class="not-prose">&nbsp; spec_json = jsonencode(</code><br /><code class="not-prose">&nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "triggers" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "startStepNames" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "CheckMonitorStatus"</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ],</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "monitorTrigger" : {}</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; ],</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "steps" : concat([</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "CheckMonitorStatus",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "actionId" : "com.datadoghq.core.if",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "parameters" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "joinOperator",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "value" : "or"</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "conditions",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "value" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "comparisonOperator" : "eq",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "leftValue" : "",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "rightValue" : "Alert"</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; },</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "comparisonOperator" : "eq",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "leftValue" : "",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "rightValue" : "Warn"</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ]</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ],</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "outboundEdges" : local.check_monitor_outbound_edges,</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "display" : {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "bounds" : {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "x" : 0,</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "y" : 0</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; ], local.notification_steps),</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "handle" : local.sample_workflow_name,</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "connectionEnvs" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "env" : "default",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "connections" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "connectionId" : local.aws_connection_id,</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "label" : local.aws_connection_label</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ]</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; ],</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; "inputSchema" : {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; "parameters" : [</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; {</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "name" : "MonitorURL",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "type" : "STRING",</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "defaultValue" : "sample_workflow"</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; &nbsp; ]</code><br /><code class="not-prose">&nbsp; &nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; &nbsp; }</code><br /><code class="not-prose">&nbsp; )</code><br /><code class="not-prose">}</code></p>
<h2><strong>Critical Pitfalls to Avoid</strong></h2>
<h3><strong>1. Permissions &amp; Ownership</strong></h3>
<p><strong>Pitfall:</strong> Workflows created via Terraform are owned by Terraform, not by individual users. Standard-role users cannot publish or edit them directly. While this is not necessarily a problem, it could become a blocker in some situations.</p>
<p><strong>Impact:</strong> Teams may be unable to manage workflows after they’re deployed.</p>
<p><strong>Solution:</strong> Have an admin grant users permission to publish/unpublish workflows.</p>
<h3><strong>2. Step Function Limitations</strong></h3>
<p><strong>Pitfall:</strong> Datadog workflows currently have limited capabilities with Express Step Functions.</p>
<p><strong>Impact:</strong> Workflow may fail on step functions-related steps.</p>
<p><strong>Solution:</strong> Prefer Standard Step Functions and document this limitation for your team.</p>
<h3><strong>3. Datastore Management</strong></h3>
<p><strong>Pitfall:</strong> Datastores cannot be provisioned via Terraform, and workflows need explicit permissions to access them.</p>
<p><strong>Impact:</strong> Workflow may fail due to missing datastore access.</p>
<p><strong>Solution:</strong></p>
<ul>
<li>Create your datastore in the Datadog console and reference it by ID in Terraform.</li>
<li>Grant Terraform a <strong>Manager</strong> role in the datastore:
<ol>
<li>Go to the Datastore page in Datadog.</li>
<li>Click the <span style="color: #ff0201;"><code class="not-prose">Settings</code></span> icon.</li>
<li>Select <span style="color: #ff0201;"><code class="not-prose">Edit Permissions</code></span></li>
<li>Add Terraform with Manager role</li>
</ol>
</li>
</ul>
<h3><strong>4. AWS Connection Management</strong></h3>
<p><strong>Pitfall:</strong> AWS connections are environment-specific and must be pre-created in Datadog.</p>
<p><strong>Impact:</strong> Workflow will fail if connections are missing or incorrect.</p>
<p><strong>Solution:</strong></p>
<ul>
<li>Create AWS connections for each environment in the Datadog console and reference them by ID in Terraform.
<ol>
<li>Click<span style="color: #ff0201;"> <code class="not-prose">Actions</code> </span>in the Datadog sidebar.</li>
<li>Select <span style="color: #ff0201;"><code class="not-prose">Connections</code></span></li>
<li>Click on the <span style="color: #ff0201;"><code class="not-prose">+ New Connection</code></span> button at the top of the connections page.</li>
<li>Select <span style="color: #ff0201;"><code class="not-prose">AWS</code></span> from the list of possible integrations.</li>
<li>Complete the form, then copy the generated IAM policy statement (the statement only shows at creation of the connection).</li>
<li>In the AWS console, create a new role and attach the generated policy statement to it.</li>
</ol>
</li>
</ul>
<h2><strong>Benefits Achieved</strong></h2>
<ol>
<li><strong>Team Autonomy</strong>: Each team can own and modify their workflow variations independently.</li>
<li><strong>Environment Consistency</strong>: Identical workflows can be deployed across environments with environment-specific configurations.</li>
<li><strong>Infrastructure as Code</strong>: Workflows are now part of the infrastructure, enabling automated deployments and rollbacks.</li>
<li><strong>Documentation</strong>: The Terraform code serves as living documentation of the workflow logic.</li>
</ol>
<p>Terraform support for Datadog workflows is a game-changer. It brings version control, consistency, and automation to what used to be a manual, UI-driven process. While the JSON structures can be complicated to set up, the long-term benefits outweigh the initial setup cost.</p>
<h2><strong>Further Reading</strong></h2>
<p>For more details on Datadog workflows, here are some resources:</p>
<ul>
<li><a href="https://registry.terraform.io/providers/DataDog/datadog/latest/docs/resources/workflow_automation">Datadog Workflow Automation Terraform Resource</a></li>
<li><a href="https://registry.terraform.io/providers/DataDog/datadog/latest/docs">Terraform Datadog Provider Documentation</a></li>
<li><a href="https://docs.datadoghq.com/workflows">Datadog Workflow Automation Guide</a></li>
</ul>
<p><!-- notionvc: 91a3c695-34ec-4c70-95fa-47f1c1242598 --></p>
<p><!-- notionvc: 2f4fdcd6-35c8-4159-9ce3-599991ec474f --></p>
<p><!-- notionvc: 957fb945-a78c-4358-a84b-8b8e9c581ba0 --></p>
<p>The post <a href="https://runa.io/blog/terraforming-datadog-workflows/">Terraforming Datadog Workflows</a> appeared first on <a href="https://runa.io">Runa</a>.</p>
