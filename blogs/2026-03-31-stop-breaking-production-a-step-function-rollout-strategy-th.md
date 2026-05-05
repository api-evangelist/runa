---
title: "Stop Breaking Production: A Step Function Rollout Strategy That Works"
url: "https://runa.io/blog/stop-breaking-production-a-step-function-rollout-strategy-that-works/"
date: "Tue, 31 Mar 2026 13:53:19 +0000"
author: "Anita Ernszt"
feed_url: "https://runa.io/feed"
---
<p>How confident are you that your next change won’t blow up production?</p>
<p>In traditional server-based systems, gradual rollouts and canary deployments are standard practice. In serverless systems, however, we too often treat infrastructure as magic black boxes rather than critical, complex systems that deserve the same rigor. Even with extensive pre-production testing, this leaves us exposed to unexpected outages.</p>
<p>Gradual rollouts are not a “nice-to-have” — they’re essential, even for serverless.<!-- notionvc: b7476352-aed3-4efb-995c-c22ef459d7da --><span id="more-27440"></span></p>
<h3><span class="discussion-id-303c1d69-b5af-804d-84c6-001c31a53854 notion-enable-hover">Why Gradual Rollouts Matter To Us</span><!-- notionvc: e5151190-7bf3-45d1-8c51-22e7d83d3b58 --></h3>
<p>Runa embarked on its serverless journey nearly 3 years ago, adopting serverless technologies to power our next-generation global payout platform. The platform needs to be fast, but reliability is non-negotiable.</p>
<p>At Runa, this problem is not theoretical. We use Step Functions extensively across our platform, and they sit on the path of many of our most important production workflows. That scale is what made rollout safety impossible to ignore. Engineers ship changes several times a day. Every release introduces risk, and without guardrails, that risk quickly compounds. To maintain high deployment velocity without compromising availability, we needed a deployment mechanism that ensured:<!-- notionvc: ccf1573c-b898-479f-ab95-6f91309e4838 --></p>
<ul>
<li>Smooth end-user experience during releases<!-- notionvc: da570641-4795-43a9-97e8-67e2407d55ec --></li>
<li>Real-time monitoring and automated rollback capabilities<!-- notionvc: 724ee21c-a581-4dfc-b53f-8a28cf543065 --></li>
</ul>
<p>AWS already gives Lambda users a familiar deployment model via CodeDeploy, with built-in traffic shifting, and automated rollback. We wanted the equivalent operational model for Step Functions in the Terraform-driven environment we actually run. That is what led us to build Tweety.<!-- notionvc: 97369539-35f7-4bf2-974d-60ca41d39412 --></p>
<h3>Defining Requirements</h3>
<p>Before designing the rollout mechanism, we aligned on a small set of baseline requirements. These were not aspirational goals. They were constraints driven by how we build and operate production systems:<!-- notionvc: ac66eba0-bc7d-4193-88ff-4d8e9bff50b5 --></p>
<ul>
<li><span class="notion-enable-hover">Terraform Compatibility: Our infrastructure is managed via Terraform. The rollout solution had to integrate seamlessly, leveraging the tools our engineers already use.</span><!-- notionvc: 7e1acf0e-ac91-4148-baa9-661cc3503a21 --></li>
<li>Automation and Guardrails: Rollout and rollback processes had to be fully automated with built-in safeguards to prevent accidental missteps.<!-- notionvc: 0dcd019e-84ee-4522-9576-a42cfe92b9ef --></li>
<li>Simplicity for Engineers: The underlying implementation could be complex, but interacting with it should not be. From an engineer’s perspective, deploying with gradual rollouts needed to feel no different from a standard release.</li>
</ul>
<h3>Planning For A Rollout</h3>
<p>Before we could even start thinking about gradual releases, we needed two critical capabilities:<!-- notionvc: 046254c3-1528-4f50-af77-b95efaca5976 --></p>
<ul>
<li>The ability to version your changes</li>
<li>The ability to balance traffic between the versions</li>
</ul>
<p>A gradual rollout only works if we can precisely control which version of a Step Function is executed at any given time. AWS Step Functions already support versioning (publishing a version creates an immutable snapshot of the state machine definition). Using aliases, we can then control which version is invoked and gradually shift traffic between versions.</p>
<p>In AWS, invoking a specific version is achieved using qualified ARNs, which explicitly reference either a published version or an alias.<!-- notionvc: d07d9bc6-ed49-4935-a9fd-db9a360f5487 --></p>
<p>For example:</p>
<ul>
<li><strong>Unqualified ARN</strong> – always points to the latest version:
<pre><code>arn:aws:states:us-east-1:123456789012:stateMachine:OrderProcessor
</code></pre>
</li>
<li><strong>Version Qualified ARN</strong> – explicitly points to a published version:
<pre><code>arn:aws:states:us-east-1:123456789012:stateMachine:OrderProcessor:3
</code></pre>
</li>
<li><strong>Alias Qualified ARN</strong> – points to an alias that can route traffic between versions:
<pre><code>arn:aws:states:us-east-1:123456789012:stateMachine:OrderProcessor:prod
</code></pre>
</li>
</ul>
<p><!-- notionvc: 7f38371d-ee64-4103-a25f-918f2f014152 --></p>
<p><!-- notionvc: 9706463d-6d3a-4e50-9b02-c14c361f50fa --></p>
<p>In order to ensure stability during the rollout, we must use qualified ARNs across our infrastructure.<!-- notionvc: c9d24a71-6b49-44c0-ba10-3d109fd838f4 --><!-- notionvc: e187e628-4de3-4df7-83f4-e775f5e5f7a3 --></p>
<h3>Only Shift Traffic At The Entry Point</h3>
<p>Because Step Functions execute downstream services as part of a single workflow, traffic shifting can only happen at the entry point, which is the alias of the top-level state machine being invoked. Any child resources, such as nested Step Functions or Lambdas, must be version pinned.</p>
<p>If we attempt to shift traffic for downstream resources independently, we risk creating version mismatches when the contract between versions changes. For example, a new state machine may invoke an older Lambda with an incompatible input, or an older state machine may call a newer Lambda expecting fields that are no longer present. These mismatches can lead to runtime errors or partial outages that are hard to detect during rollout.</p>
<p>The diagram below shows an example of an incorrect rollout, where traffic is shifted independently across state machines and downstream Lambdas:</p>
<p><img alt="" class="alignnone size-full wp-image-27441" height="1220" src="https://runaio.wpenginepowered.com/wp-content/uploads/2026/03/Screenshot-2026-03-16-at-3.53.30-pm.png" width="1356" /></p>
<h3>Version Pin Downstream Dependencies</h3>
<p>To ensure that changes to child resources are rolled out gradually and safely, we made version pinning mandatory for all downstream resources. This prevents changes from reaching production immediately and ensures that each Step Function version executes against a consistent set of dependencies.</p>
<p>Version pinning also plays a key role in driving the gradual rollout process as new versions of downstream resources are introduced and referenced by a new state machine version.<!-- notionvc: d38a5814-ff9c-42fe-869e-02eafff71ffc --></p>
<p><img alt="" class="alignnone wp-image-27442 size-full" height="1638" src="https://runaio.wpenginepowered.com/wp-content/uploads/2026/03/image-5.png" width="2048" /></p>
<h3>IAM Resources Are Not Versioned</h3>
<p>This is easy to overlook during rollouts and has caused real production failures for us in the past.</p>
<p>While Step Functions can be versioned and aliased, IAM roles and policies have no built-in versioning. All versions of a Step Function share the same IAM configuration.</p>
<p>This has an important implication. Any IAM change must remain fully backward compatible with <strong>all</strong> Step Function versions that are still receiving traffic. Removing permissions, even if they are no longer required by the latest version, can cause older executions to fail with <code>AccessDenied</code> errors during a rollout.</p>
<p>Once we were able to version workflows and shift traffic safely, we formalised 3 non-negotiable rules:</p>
<ol>
<li>Pin all downstream resources</li>
<li>Shift traffic only at the entry point</li>
<li>Keep IAM changes backward compatible</li>
</ol>
<p>Violating any of these rules risks breaking production in the middle of a rollout.<!-- notionvc: 73b385ec-c97a-4b4f-a337-a07d2a51fb03 --></p>
<h3>Meet Tweety</h3>
<p>To orchestrate the release process, we built Tweety, our in-house release manager implemented as a Step Function. The name is inspired by the Looney Tunes canary, reflecting Tweety’s role as an early warning system during deployments.</p>
<p>The idea behind Tweety is deliberately simple. If you have used Lambda canary deployments via CodeDeploy before, the model will feel familiar: publish an immutable version, shift traffic gradually, monitor health, and roll back automatically when signals degrade.</p>
<p>Tweety brings that operating model to Step Functions, packaged in a way that fits our Terraform workflows, our dependency-pinning rules, and the realities of rolling out orchestration logic safely.</p>
<p>It handles rollout between versions, monitors health checks, and automatically rolls back if alarms are triggered:</p>
<p><img alt="" class="alignnone size-full wp-image-27444" height="1226" src="https://runaio.wpenginepowered.com/wp-content/uploads/2026/03/Screenshot-2025-01-08-at-6.17.22-pm.png" width="1682" /></p>
<p>To kick off the gradual rollout process, we invoke Tweety whenever a Step Function version changes. The invocation is triggered from Terraform using the AWS CLI, with user-defined inputs such as traffic shift percentage, rollout interval, and monitoring thresholds:<!-- notionvc: 8369ef9d-301b-4d5e-8147-7cdee716cbea --></p>
<pre><code class="language-hcl">resource "null_resource" "run_gradual_rollout" {
  triggers = {
    sfn_version = var.version_arn
  }

  provisioner "local-exec" {
    # Only trigger deployment if the state machine version is not equal to 1
    command = &lt;&lt;EOT
      if [ ${var.version_arn} != ${local.current_version_arn} ]; then 
        aws stepfunctions start-execution \\
          --state-machine-arn ${var.gradual_rollout_stepfunction} \\
          --input '${jsonencode(local.rollout_payload)}'
      fi
    EOT
  }
}
</code></pre>
<p>Once the rollout is initiated, Tweety runs two jobs in parallel:</p>
<ul>
<li><strong>Traffic management</strong> This job determines the next traffic weight and updates the Step Function alias accordingly.</li>
<li><strong>Health checks</strong> In parallel, the <code>Health check</code> job continuously evaluates system metrics against team-defined thresholds. If any degradation is detected, it stops the rollout, initiates a rollback, and alerts the team that the release has failed.</li>
</ul>
<p>Building our own gradual rollout process allowed us to fully customise it to our needs, from Slack notifications to support for non-standard deployment strategies.</p>
<p><!-- notionvc: a0afa02a-a087-4099-ac35-9ce2c458bdd3 --></p>
<h3>DevX as a functional requirement</h3>
<p>Ease of adoption was a first-class concern when designing the rollout solution. Gradual rollouts had to be simple to adopt and should not require engineers to understand the internal workings of the rollout engine. To achieve this, we wrapped the required resources in a Terraform module that hides the complexity and manages the gradual rollout process end to end.</p>
<p>From an engineer’s perspective, deploying with gradual rollouts feels no different from any other change, except that it is safer. Engineers provide a small set of inputs to a Terraform module, and the rest is handled automatically: versioning, alias shifts, monitoring, rollback, and Slack updates.</p>
<p>The visual execution graphs and built-in logging provided by Step Functions also help engineers better understand the deployment process and simplify troubleshooting when things go wrong.</p>
<p><!-- notionvc: 392f22c8-dad0-4464-9d5c-3617f93a779e --></p>
<h3>Closing thoughts</h3>
<p>Once the gradual rollout mechanism was in place, it was quickly adopted across our engineering teams. A key enabler was the set of custom Terraform modules we built, which made safe rollouts easy to apply and operate consistently.</p>
<p>By adding safety guardrails to Step Functions, we enabled engineers to move fast and deploy with confidence, even during peak season, without putting production stability at risk.</p>
<p>While AWS provides powerful primitives such as Step Function versions and aliases, building a safe rollout workflow still requires stitching those pieces together with operational guardrails. Tweety is our attempt to codify those best practices into a reusable tool.</p>
<p>We are now open-sourcing the Terraform module behind Tweety so other teams can adopt and adapt the same approach in their own AWS environments.</p>
<p>To get started, you can find our opensource module here: <a href="https://registry.terraform.io/modules/wegift/tweety-sfn/aws/latest">https://registry.terraform.io/modules/wegift/tweety-sfn/aws/latest</a></p>
<p><!-- notionvc: c7fedea5-3a6a-49fc-8bb3-bc17a0d377e2 --></p>
<p><!-- notionvc: d9d5276d-1c44-4e08-82b3-52384e9f3e15 --></p>
<p>The post <a href="https://runa.io/blog/stop-breaking-production-a-step-function-rollout-strategy-that-works/">Stop Breaking Production: A Step Function Rollout Strategy That Works</a> appeared first on <a href="https://runa.io">Runa</a>.</p>
