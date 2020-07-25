---
date: 2020-07-25
title: Pruning ECR Images used on Kubernetes
tags: [
    "AWS",
    "ECR",
    "Kubernetes",
    "Go", 
]
---

In this post we will talk about taking advantage of [ECR][ECR]'s [lifecycle policies][ECR Lifecycle Policies] 
to prune images that were not deployed on [Kubernetes][Kubernetes].

We will be using [AWS][AWS], a [Kubernetes][Kubernetes] Cluster (self-hosted or [EKS][EKS]), 
as well as a [Continuous Integration][Continuous Integration] system, or CI for short, to build and push images continously to ECR.

If we just keep pushing images to ECR we will eventually hit [the maximum number of images in a given repository][ECR Limits] and 
receive an error message for every subsequent push attempt.

In most cases, especially at the beginning of a project, there won't be a problem because the maximum number of images per repository is quite high 
(it is currently set to 10000 by default. [It was increase from its previous value of 1000][ECR Limits Increase]).

For all other cases we should use ECR's lifecycle policies to, at the very least, remove untagged images and prune old images.
But, once we start using them we quickly realize that the allowed rules are quite limited. For example we cannot currently 
remove all images with a certain tag prefix older than a certain number of days while keeping the latest one even if it is too old.

To solve this we could either create our own service or tool to do that and avoid using ECR's lifecycle policies or we
could complement it with a small service or tool that adds a given tag to certain images.

I went with the later and wrote a small tool called [kube-ecr-tagger][kube-ecr-tagger] for that.

Before presenting the proposed solution, let's have a look into ECR lifecycle policies a bit more in detail.

## ECR Lifecycle Policies
 
[ECR lifecycle policies][ECR Lifecycle Policies], as written on its page, is a part of ECR that enables us to specify 
the lifecycle management of images in a repository. 
A lifecycle policy, such as the following, is a set of one or more rules, where each rule defines an action for ECR. 
The actions apply to images that contain tags prefixed with the given strings. 
This allows the automation of cleaning up unused images, for example expiring images based on age or count.

```json
{
    "rules": [
        {
            "rulePriority": 1,
            "description": "Expire images older than 14 days",
            "selection": {
                "tagStatus": "untagged",
                "countType": "sinceImagePushed",
                "countUnit": "days",
                "countNumber": 14
            },
            "action": {
                "type": "expire"
            }
        }
    ]
}
```

The allowed rules and actions are limited. They do not allow complex rules such as remove all images with a given tag prefix after a certain amount of time
while also excluding the latest one.

There cannot be exceptions to a rule. Once a rule matches an image it cannot be changed by another rule with a lower priority.
You can only add and not remove from a rule.

That's why we need something more in order to solve our problem.

## kube-ecr-tagger

To keep the lifecycle policies simple and to ensure that no unnused image remains in our ECR repositories, I wrote a simple tool called [kube-ecr-tagger][kube-ecr-tagger].

The tool runs inside the cluster and tags the used ECR images in a given namespace, or in all namespaces, with either a given tag or a tag created by appending a unix timestamp to the passed tag prefix.

If a tag is passed then there will only be one such tag in each repository. If a tag prefix is passed instead then there will be multiple tags in each repository with the same prefix.

Let's use a concrete example. Let's assume that we use semantic versioning to tag our images in CI and that we are currently at version 1.0.X and that not
all images that are pushed will be used in production. Perhaps some of them will fail in the acceptance/staging phase or be flagged for security issues if image scanning is activated.

We want to keep the last 1000 images that are deployed in production and at the same time remove images whose tag starts with '1.0' and that are older than 30 days. For that we could the following lifecycle policy:

```json
{
    "rules": [
        {
            "rulePriority": 10,
            "description": "Keep last 1000 images with tag prefix 'production' ",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["production"],
                "countType": "imageCountMoreThan",
                "countNumber": 100
            },
            "action": { "type": "expire" }
        },
        {
            "rulePriority": 20,
            "description": "Remove images with tag prefix '1.0' that are older than 30 days",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["1.0"],
                "countType": "sinceImagePushed",
                "countNumber": 30,
                "countUnit": "days"
            },
            "action": { "type": "expire" }
        }
    ]
}
```

Then, in order to tag the images in production, which we assume for the sake of simplicity are deployed in the **prod** namespace, we will deploy kube-ecr-tagger with the following command:

```bash
kube-ecr-tagger --namespace=prod --tag-prefix=production
```

Of course, we have to make sure that it has the right IAM permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:DescribeImages",
                "ecr:BatchGetImage",
                "ecr:PutImage",
            ],
            "Resource": "*"
        }
    ]
}
```


[AWS]: https://aws.amazon.com/
[EKS]: https://aws.amazon.com/eks/
[ECR]: https://aws.amazon.com/ecr/
[ECR Limits]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html
[ECR Limits Increase]: https://aws.amazon.com/about-aws/whats-new/2019/07/amazon-ecr-now-supports-increased-repository-and-image-limits/
[ECR Lifecycle Policies]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html
[Kubernetes]: https://kubernetes.io/
[Continuous Integration]: https://en.wikipedia.org/wiki/Continuous_integration
[kube-ecr-tagger]: https://github.com/AnesBenmerzoug/kube-ecr-tagger
