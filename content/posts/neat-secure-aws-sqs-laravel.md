---
title: "Neat & Secure: Adding AWS SQS to a Laravel 5.6 Application"
date: 2020-10-16T23:14:12Z
draft: false
---

# Neat & Secure: Adding AWS SQS to a Laravel 5.6 Application

Y'know sometime when you're working on an old codebase? and you wanna do something "new" but the docs don't really help you much? Yeah?

That's what I'm documenting here.

So I'm working on getting a Laravel 5.6 application into AWS Fargate with Terraform. This application began life as a Laravel 5.0 app ~6 years ago, so some of the codebase was somewhat templated from then. I’d like so add, I’m not so much a software developer and not a PHP one at that. More a gluer-together-of-parts-er, on of those “DevOps” you hear of. So this is part guide part journey.

## Making changes
First up I made changes to the config, `config/queue.php`.  

```diff
         'sqs' => [
             'driver' => 'sqs',
-            'key' => 'your-public-key',
-            'secret' => 'your-secret-key',
-            'prefix' => 'https://sqs.us-east-1.amazonaws.com/your-account-id',
-            'queue' => 'your-queue-name',
-            'region' => 'us-east-1',
+            'key' => env('AWS_ACCESS_KEY_ID'),
+            'secret' => env('AWS_SECRET_ACCESS_KEY'),
+            'prefix' => env('SQS_PREFIX'),
+            'queue' => env('SQS_QUEUE'),
+            'region' => env('AWS_REGION'),
         ],
```

This is the config after some iteration. As we’re running this in a container, it’s incredibly useful to pass config through via environment variables. [The Twelve-Factor App](https://12factor.net) methodology really chimes with me, the 3rd point is about config and passing config via the environment, decoupling your app from it’s runtime environment since things will vary between staging, production, local dev environments.

## Don’t forget your dependencies
I skimmed the Docs a bit too quick and missed out that I needed to include the AWS SDK as it wasn’t already included.

```shell
composer require aws/aws-sdk-php
```

## What’s Terraform doing?
So in Terraform we can create a simple SQS Queue with the following code:

```terraform
resource "aws_sqs_queue" "myqueue" {
  name = "myQueue"
  ...
}
```

Then we can get the SQS Queue URL and pass this into our Fargate Task Definition. This is an excerpt and will need to be passed to a `aws_ecs_task_definition` resource.

```json
{
    "family": "",
    "containerDefinitions": [
        {
            "name": "",
            "image": "",
            ...
            "environment": [
                {
                    "name": "SQS_QUEUE",
                    "value": "${aws_sqs_queue.myqueue.id}"
                }
            ],
            ...
        }
    ],
    ...
}
```

### What did I learn here?

So if you go back and read the diff of `config/queue.php` you’ll see that they had the prefix as `https://sqs.us-east-1.amazonaws.com/your-account-id` and the queue name as `your-queue-name`. I initially was passing `aws_sqs_queue.myqueue.id` to `SQS_PREFIX`and `aws_sqs_queue.myqueue.name` to `SQS_QUEUE`. But this resulted in the application getting the full queue as `https://sqs.us-east-1.amazonaws.com/your-account-id/myQueue/myQueue`.

Long story short, you can forget about `prefix (SQS_PREFIX)` and just pass the full url to `queue (SQS_QUEUE)`.

The `aws_sqs_queue`resource, the `id`returns a URL, which isn’t imidietly obvious. As I use Intelij IDEA to write Terraform it wasn’t clear what `id`would return. Second reading of the docs makes this slightly clearer.

There is a [PR](https://github.com/terraform-providers/terraform-provider-aws/issues/11848) in to add an output of `url` to the resource’s outputs which should clear this up.


## Where’s the access key and secret?
One thing I really love about how IAM is integrated into the rest of the AWS Ecosystem is the concept of Instance Profiles, or when you’re using Fargate these are Task Roles.

Which means we don’t have to setup any credentials as the AWS PHP SDK will look for creds in a few places. You can read more here: [Credentials for the AWS SDK for PHP Version 3 - AWS SDK for PHP](https://docs.aws.amazon.com/sdk-for-php/v3/developer-guide/guide_credentials.html)

What we do need to do is create a task role and attach a policy, I’ll not go super into detail but the policy will look something like this:

```terraform
resource "aws_iam_policy" "sqs" {
  name        = "sqs-myQueue-poicy-rw"
  path        = "/"
  description = "myQueue: Full Access Policy"

  policy = <<-EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "",
          "Effect": "Allow",
          "Action": [
            "sqs:SendMessage",
            "sqs:ReceiveMessage",
            "sqs:GetQueueAttributes",
            "sqs:ChangeMessageVisibility",
            "sqs:DeleteMessage"
          ],
          "Resource": "${aws_sqs_queue.myqueue.arn}"
        }
      ]
    }
    EOF
}
```

Once again you can see that we’ve used Terraform variables to pass in an output from our SQS resource. This is to scope full permissions down to just this queue. 

**Teach your teams IAM!** Your IAM policies should not be full of `*`’s. This is one thing that really stuck with me from a talk that [The Scale Factory](https://www.scalefactory.com/services/secure-comply/) presented.
