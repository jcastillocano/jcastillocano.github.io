---
layout: post
title:  "When you made things more complicated than they were"
date:   2020-06-20 14:08:39 +0000
categories: terraform cloudwatch aws
---

These days I'm helping the team at [TrillerCo](https://triller.co) to integrate [Infrastructure as Code](https://en.wikipedia.org/wiki/Infrastructure_as_code) with [Terraform](https://www.terraform.io/), an old friend of mine. All the infra is hosted in AWS, so nothing new for me, but this time we started with the idea of adding all new resources in Terraform as well as importing old ones as we go (we have a mix of legacy and vanilla infra). I't been a while since I've been deeply involved in Terraform development (last time I was using terraform 0.11) so I was keen to start with all the new functionalities that were added in 0.12: new syntax, type system, iterators (yayyy!) among others. So let's go to the nitty gritty.

## First chapter: what I wanted vs what I got

Thinking about how we are going to architech our terraform modules, and having into account current infra as well as upcoming one, we decided to go with a base approach of one repo per project, and some modules for common stuff (monitoring, etc). Something like this:

 * PROJECT-X-terraform: terraform resources.
 * PROJECT-Y-terraform: terraform resources.
 * ...
 * SRE-terraform: aux modules, ci modules.

We will use external modules for monitoring, to not reinvent the wheel. First module we wanted to make usage of was the official [AWS CloudWatch](https://github.com/terraform-aws-modules/terraform-aws-cloudwatch) module for alerting, since most of our metrics were sent to CloudWatch (more on that later). The old pal [Anton Babenko](https://github.com/antonbabenko) (when are you coming to Tenerife dude?) did a fine work supporting multi dimensions' alerts in [metric-alarms-by-multiple-dimensions](https://github.com/terraform-aws-modules/terraform-aws-cloudwatch/tree/master/modules/metric-alarms-by-multiple-dimensions) module, that was exactly what we were looking for! So let's try it out:

```hcl

module "metric_alarm_msk_disk_usage" {
  source  = "terraform-aws-modules/cloudwatch/aws//modules/metric-alarms-by-multiple-dimensions"
  version = "~> 1.2"

  alarm_name          = "msk-disk-used"
  alarm_description   = "Kafka disk usage"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  threshold           = 85
  period              = 120
  unit                = "Percent"

  namespace   = "AWS/Kafka"
  metric_name = "KafkaDataLogsDiskUsed"
  statistic   = "Average"

  dimensions =  {
    "broker1" = {
      "Cluster Name" = aws_msk_cluster.msk.cluster_name
      "Broker ID" = 1
    },
    "broker2" = {
      "Cluster Name" = aws_msk_cluster.msk.cluster_name
      "Broker ID" = 2
    },
    "broker2" = {
      "Cluster Name" = aws_msk_cluster.msk.cluster_name
      "Broker ID" = 3
    }
  } 

  alarm_actions = [aws_sns_topic.pagerduty-prod.arn]
}
```

Ok, we got it. Three new alarms were created in CloudWatch. We could have stopped here, but we realized all these dimensions were hardcoded in the code, which means in case we need to scale our AWS MSK cluster we will lose further alerts. Hence the first problem: make this block dynamic.

## First problem: make it dynamic

I've heard about [dynamic blocks](https://www.terraform.io/docs/configuration/expressions.html#dynamic-blocks) in Terraform 0.12, so first thought: ok, that might be what I'm looking for. I had a look at the doc, to see what I need to do in order to make my block dynamic, and I made these changes to my code. My first try was something like this:

```hcl
  ...
  statistic   = "Average"

  dynamic "brokers" {
    for_each = range(1, var.broker_nodes + 1)
    content {
      dimensions = {
        "Cluster Name" = aws_msk_cluster.msk.cluster_name
        "Broker ID" = dimensions.value
      }
    }
  }
  ...
```

Obviously it didn't work.

```hcl
Error: Unsupported block type

  on alarms.tf line 17, in module "metric_alarm_msk_disk_usage":
  17:   dynamic "brokers" {

Blocks of type "dynamic" are not expected here.
```

Interesting. It says **Blocks of type "dynamic" are not expected here.**. Makes sense, reading the doc again I found this quote:

> You can dynamically construct repeatable nested blocks like setting using a special dynamic block type, which is supported inside resource, data, provider, and provisioner blocks

Ok, no way we use dynamics blocks for this. Let's try something else.

Lesson 1: READ THE DOCS (but fooling around is also learning) 

![Read the docs](/readthedocs.jpg)

## Wait, what now?

Ok, what else we have in Terraform for building dynamics blocks? Ah yes, locals! We will make use of several functions, like [formatlist](https://www.terraform.io/docs/configuration/functions/formatlist.html), [range](https://www.terraform.io/docs/configuration/functions/range.html), [flatten](https://www.terraform.io/docs/configuration/functions/flatten.html) and [zipmap](https://www.terraform.io/docs/configuration/functions/zipmap.html)

```hcl
locals {
  broker_names = formatlist("broker%s", range(1, var.broker_nodes + 1))
  broker_values = flatten([
    for broker_id in range(1, var.broker_nodes + 1): {
      "Cluster Name" = aws_msk_cluster.msk.cluster_name
      "Broker ID" = broker_id
    }
  ])
  broker_dimensions = zipmap(local.broker_names, local.broker_values)
}

module "metric_alarm_msk_disk_usage" {
  ...
  dimensions = local.broker_dimensions
  ...
```

Yay! It worked as expected, so that's good news. 

```hcl
Plan: 3 to add, 0 to change, 0 to destroy.
```

We have a way now to create alarms dynamically based on the number of cluster nodes. Awesome. We could rest tonight.

## Second chapter: modularize

Next day we started to add more alarms to our PROJECT-X as we did the day before, and soon we realized we will mostly have the same config across all PROJECTs, so rather than repeating the code (we attempt to follow DRY principles, yeah) everywhere we decided to create a module for monitoring MSK clusters in AWS with the basics (disk, cpu, offline partitions, etc). So far so good, let's create it.

```hcl
### Module code
locals {
  broker_names = formatlist("broker%s", range(1, var.broker_nodes + 1))
  broker_values = flatten([
    for broker_id in range(1, var.broker_nodes + 1) : {
      "Cluster Name" = var.cluster_name
      "Broker ID"    = broker_id
    }
  ])
  broker_dimensions = zipmap(local.broker_names, local.broker_values)
}

resource "aws_cloudwatch_metric_alarm" "disk-usage" {
  ...
  dimensions = local.broker_dimensions
  ...
}
```

then we call the msk module as follow

```hcl
module "msk_alarms" {
  ...
  cluster_name = aws_msk_cluster.msk.cluster_name
  broker_nodes = var.msk_number_of_broker_nodes
  ...
}
```

Fine, right? Let's push it to the CI/CD, we are not going to even test it in local (at least we have a _terraform fmt_ pre commit configured in local, which says all ok). After a while, we get the notification in slack complaining as follow:

```hcl
Error: Incorrect attribute value type

  on .terraform/modules/msk_alarms/main.tf line 33, in resource "aws_cloudwatch_metric_alarm" "disk-usage":
  33:   dimensions = local.broker_dimensions
    |----------------
    | local.broker_dimensions is object with 3 attributes

Inappropriate value for attribute "dimensions": element "broker1": string
required.
```

After a while reviewing we didn't make any typo, we stopped ourselves to understand better the message. **Inappropriate value for attribute "dimensions": element "broker1": string required.** Interesting, we are using the same code than we used for the AWS CloudWatch module, but it's failing now. 

Having a look at [terraform cloud watch resource](https://www.terraform.io/docs/providers/aws/r/cloudwatch_metric_alarm.html#dimensions), all the examples have one or more conditions (Filter = Value) but there are no examples with maps like the one we're building in *local.broker_dimensions*. That makes more sense. Then the error must be how we moved from the AWS CloudWatch module to our custom module.

But wait, we were using a different example as source! AWS CloudWatch provides two modules, [metric-alarm](https://github.com/terraform-aws-modules/terraform-aws-cloudwatch/blob/master/modules/metric-alarm) and [metric-alarms-by-multiple-dimensions](https://github.com/terraform-aws-modules/terraform-aws-cloudwatch/tree/master/modules/metric-alarms-by-multiple-dimensions), and we were looking at the wrong one! Instead of *metric-alarm*, where the code is intended for just one alarm resource, we should be looking at *metric-alarms-by-multiple-dimensions*. 

Now I see it, we are missing *for_each*! We need to iterate over each dimension to create a given resource, that's why our module is failing. Let's fix it.

```hcl
resource "aws_cloudwatch_metric_alarm" "disk-usage" {
  for_each = local.broker_dimensions
  ...
  dimensions = each.value
  ...
```

It seems to work now! We have three resources to be added, and when we run apply it creates the three of them successfully. Let's go to celebrate it! But...

![not today](/nottoday.jpg)

## Last chapter: ALWAYS check (don't trust 100% terraform)

Interesting. Another colleage opens a pull request with his changes, but not related to this, and there is one change to be applied in our previously created cloud watch alarms. Weird. Let's check again what the previous execution in Jenkins did.

Plan says `Plan: 3 to add, 0 to change, 0 to destroy.`. Checked.

Apply says `Apply complete! Resources: 3 added, 0 changed, 0 destroyed.`  Checked.

Another poltergeist. Great. It's thursday afternoon, give me a break!

We go to the cloud watch aws console to see what is actually there, but for my surprise there is just... one alarm! But if terraform said it creates three of them, what's going on? Mmmm, something is not right, but I cannot find what it is. Terraform code seems right, I run the apply again and... tada!

### How to use for_each at top level

Resource name! All three resources are being created with the same name, so the next one overwrites the previous one! But terraform didn't complain, interesting... 

![facepalm](/facepalm.png)

This means we just need to add *${each.key}* in the name property to properly identify each resource with its broker index. Done.

```hcl
resource "aws_cloudwatch_metric_alarm" "disk-usage" {
  for_each   = local.broker_dimensions
  alarm_name = "${var.project}-msk-disk-used-${each.key}-${var.environment}"
  ...
```

Plan, apply, ready! Checking again AWS CloudWatch console, there we have our three alarms! Weeeee! 

## What we've learnt today?

Firstable, read the docs. When you're testing or trying a new feature, piece of code, module, etc, please please please read the docs! 

Also, do not forget to use the right source code when you're checking against upstream repos. Using a different source file could save you hours of coding!

And last but not least, failing is good. You have no idea how many things I've learnt during this session, that's a lesson I'll never forget. Good night.
