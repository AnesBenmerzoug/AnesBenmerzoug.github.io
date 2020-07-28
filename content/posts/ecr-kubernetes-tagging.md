---
date: 2020-07-28
title: When ECR lifecycle policies are not enough...
tags: [
    "AWS",
    "ECR",
    "Kubernetes",
    "Go", 
]
---

In this second post we will talk about something a bit different from last time.
we will be using a tool to extend and take full advantage of [ECR][ECR] [Lifecycle Policies][ECR Lifecycle Policies] 
to prune more images than is possible with just lifecycle policies.

We will be using [AWS][AWS], a [Kubernetes][Kubernetes] Cluster (self-hosted or [EKS][EKS]),
as well as a [Continuous Integration][Continuous Integration] system, or CI for short, to build and push images continously to ECR.

Let's assume that we are continously building and pushing images to ECR after each push to the master branch.
If we just keep doing that we will eventually hit [the maximum number of images in a given repository][ECR Limits] and 
receive an error message for every subsequent push attempt.

In most cases, especially at the beginning of a project, there won't be a problem because the maximum number of images per repository is quite high 
(the limit used to be 1000 [but was recently increased to 10000][ECR Limits Increase]).

For all other cases it is recommended to use ECR Lifecycle Policies to, at the very least, remove untagged images and prune very old images.
But, once we start using them we quickly realize that the allowed rules are quite limited. For example we cannot currently 
remove all images with a certain tag prefix older than a certain number of days while keeping the last **N** even if they are too old.

Here are some examples of rules that cannot be created using them:

- Remove all images with a certain tag prefix that are older than a certain number of days while keeping the last **N** even if they are old:
    
    This can happen if, for example, development stops on a certain project for some time i.e. no new images are being pushed but at the same
    time it is still deployed in production.

- Keep all images in one repository that have a tag that already exists in a second repository:

    This can happen if, for example, we have two different components i.e. two images that require the same version i.e. same tag to run properly.

- Keep the last **N** images that were deployed to production and remove all other ones after some time:

    This can happen if, for example, not all images that are pushed to ECR are used in production. It can because of errors in the **Staging** phase or security vulnerabilities, etc. 

In this post we will focus on solving the problem from the last example.

To do so we could either create our own service/tool to directly prune images and avoid using ECR Lifecycle Policies or we
could complement it with a small service/tool that adds a given tag to certain images.

I went with the later and wrote a small service called [kube-ecr-tagger][kube-ecr-tagger] that runs inside the cluster and tags images for us.

Before presenting it, let's have a look into ECR Lifecycle Policies a bit more in detail.

## ECR Lifecycle Policies
 
[ECR lifecycle policies][ECR Lifecycle Policies] are a part of ECR that enables us to specify 
the lifecycle management of images in a repository. 
A lifecycle policy, such as the following, is a set of one or more rules, where each rule defines an action for ECR. 
The actions apply to images that contain tags prefixed with the given strings. 
This allows the automation of cleaning up unused images, for example expiring images based on age or count:

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

The allowed rules and actions are limited. For example, you cannot do any of the following:

- Match the same image with multiple rules ( this could be used to add exceptions to a rule ). If a rule matches an image it cannot be matched by another rule with a lower priority
- Expire images both by count and by age
- Choose to keep images that match a rule instead of expiring them
- Match images with an exact tag, only tag prefixes are allowed

To avoid some of these limitations and to solve our problem from the previous section I developed a simple tool called [kube-ecr-tagger][kube-ecr-tagger].

## kube-ecr-tagger

[kube-ecr-tagger][kube-ecr-tagger] runs inside the cluster and tags the used ECR images in a given namespace, or in all namespaces, with either a given tag or a tag created by appending a unix timestamp to the passed tag prefix.

If a tag is passed then there will only be one such tag in each repository. If a tag prefix is passed instead then there will be multiple tags in each repository with the same prefix.

Let's use a concrete example. Let's assume that we use semantic versioning to tag our images in CI and that we are currently at version 1.0.X and that not
all images that are pushed will be used in production. Perhaps some of them will fail in the acceptance/staging phase or be flagged for security issues if image scanning is activated.

We want to keep the last 100 images that are deployed in production and at the same time remove images whose tag starts with '1.0' and that are older than 30 days. For that we could the following lifecycle policy:

```json
{
    "rules": [
        {
            "rulePriority": 10,
            "description": "Keep last 100 images with tag prefix 'production' ",
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

Then, in order to tag the images in production, which we assume for the sake of simplicity are deployed in the **prod** namespace, we will deploy kube-ecr-tagger as a Deployment in the **kube-system** namespace with the following example manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
   name: kube-ecr-tagger
   namespace: kube-system
spec:
   template:
      spec:
        containers:
         - name: kube-ecr-tagger
           image: anesbenmerzoug/kube-ecr-tagger:latest 
           command:
           - kube-ecr-tagger
           args:
           - --namespace=prod
           - --tag-prefix=production
```

We should not forget to add a service account, if we're using IAM roles for service accounts, or the right annotation, if we're using [kiam][kiam] or [kube2iam][kube2iam], with the right IAM permissions:

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

kube-ecr-tagger will then check ECR images of all deployed containers in the **prod** namespace and add tags with the prefix **production** if it is not already present.

## Conclusion

We have seen in this post that by combining a small tool with ECR Lifecycle Policies we can achieve more complicated rules than with plain lifecycle policies.

Of course, [kube-ecr-tagger][kube-ecr-tagger] is limited in what it can do but one can already have an idea of what can be achieved.

If you want to try kube-ecr-tagger you can simply
use the built container images from [this repository](https://hub.docker.com/r/anesbenmerzoug/kube-ecr-tagger) on Dockerhub.

I hope that you have learned at a thing or two from this post. 
If there are any mistakes or if you have questions please do not hesitate to reach out to me.


[AWS]: https://aws.amazon.com/
[EKS]: https://aws.amazon.com/eks/
[ECR]: https://aws.amazon.com/ecr/
[ECR Limits]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/service-quotas.html
[ECR Limits Increase]: https://aws.amazon.com/about-aws/whats-new/2019/07/amazon-ecr-now-supports-increased-repository-and-image-limits/
[ECR Lifecycle Policies]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html
[Kubernetes]: https://kubernetes.io/
[Continuous Integration]: https://en.wikipedia.org/wiki/Continuous_integration
[kube-ecr-tagger]: https://github.com/AnesBenmerzoug/kube-ecr-tagger
[kiam]: https://github.com/uswitch/kiam
[kube2iam]: https://github.com/jtblin/kube2iam
