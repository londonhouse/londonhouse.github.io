---
title: "Securing Amazon SageMaker, Part 2: Domain User Authentication"
date: 2026-11-01 09:00:00 -0500
seo_title: "Securing SageMaker, Part 2: Studio User Authentication"
description: "Separating SageMaker domains by team, choosing IAM or SSO, and using tag-based IAM conditions to stop users opening each other's Studio user profiles."
tags: [AWS, SageMaker, Security, IAM, IAM Identity Center, Access Control, Terraform]
image: /assets/images/sagemaker-studio-access-checklist.png
---

**How do you control who can open SageMaker Studio, and what they can do once they're in?** Separate your domains by team, pick IAM or SSO authentication when you create the domain (you can't change it later), and tag each user profile so a condition on the user's sign-in role only lets them open their own profile. Once they're inside Studio, every action runs as the user profile's execution role, so that's where data access like S3 belongs.

[![Five-step checklist for locking down SageMaker Studio access: 1 map domains per team, 2 choose IAM or SSO auth at domain creation, 3 lock profiles with a userId tag and an aws:userid condition, 4 deny opening Studio outside your VPC endpoint, 5 scope an execution role per user profile](/assets/diagrams/sagemaker-studio-access-checklist.svg)](/assets/diagrams/sagemaker-studio-access-checklist.svg "Open the full-size diagram")

So, in [Part 1](/articles/securing-sagemaker-vpc-only-mode/) we discussed securing your SageMaker Domain deployment by hardening your network. To quickly reiterate, we discussed:

- Deploying a private subnet
- Enabling your VPC to use private DNS:
    - Set `enable_dns_support` and `enable_dns_hostnames` to `true`
- Deploying VPC endpoints with security groups
- Configuring SageMaker Domain to be VPC-only

When you have these settings completed, SageMaker communicates traffic over the AWS cloud's network without traversing the public internet, all thanks to the magic of DNS resolution.

The second level of security I want to cover is user authentication. However, before we get into that, let's discuss what the users are authenticating into.

## What are users authenticating into? Domains, user profiles, and roles

Part 1 introduced SageMaker Domain, Studio, and Spaces as logical separations. Generally, I like to map domains with respect to the teams using them. If you have multiple analytical teams or business units looking to use SageMaker Studio, you can deploy different domains to each team or BU.
This creates a grouping-like structure that aids administration and IAM policy separation, for things like:

- Outside SageMaker AWS Resources (EMR, AWS ML/AI services)
- S3 access for cross-account bucket access.

SageMaker allows you to think of IAM role access in 3 different ways:

- **Sign-in role**
- **User profile execution role**
- **Domain default execution role**

The key thing to understand is that the person signing in and the Studio user are not the same user identity.

The **sign-in role** is your user's normal IAM or SSO identity, the IAM role or SSO permissions your user signs in to AWS with. Its policy controls whether they can open Studio, and which user profile they can open.

Once they're in Studio, they're no longer acting as that IAM or SSO identity. They're acting as their _"Studio User"_, which is their **user profile**, and every AWS action they take from within Studio (reading S3, launching a training job, calling EMR) uses the user profile's **execution role**, not their IAM permissions.

This article covers private spaces, where each user works on their own. Shared spaces don't use the user profile's execution role; they run as the domain's default space execution role, so they're out of scope here.

The **domain default execution role** is the fallback. Any user profile created without its own execution role uses it. I generally only put the standard guardrails in this role's policy. The **user profile execution role** is set on each user profile, and it's where fine-grained, per-user or per-team access goes.

So the first layer of user authentication protection is to map your domains by teams or business units, and to design your role assignments. Next, with that structure in place, you reference your organization's authentication policies and determine which of SageMaker Domain's authentication methods will be used to access SageMaker Studio.

## How do users authenticate to SageMaker Studio: IAM or SSO?

When you create your domain, you can choose between IAM authentication and SSO. This is important because changing your mind later means a complete redeployment.

Choosing either method only depends on your shop's preferred authentication method. If you're already using SSO with external IdPs, then SSO is probably your best bet.

If you're not using an external IdP and you are planning to rely on IAM for restrictions, then standard IAM login is your way. Also note that, as of November 2026, if you're on GovCloud, you really don't have a choice but to use IAM. [SageMaker AI in AWS GovCloud (US)](https://docs.aws.amazon.com/govcloud-us/latest/UserGuide/govcloud-sagemaker.html "Amazon SageMaker AI in AWS GovCloud (US)") supports IAM authentication for Studio, but not IAM Identity Center.

One more thing to know before you choose: SSO here means a domain created in IAM Identity Center mode. Those users open Studio from the AWS access portal and land in the user profile created for them.

The profile and network restrictions below are for IAM-mode domains, and that includes users who sign in to AWS with an SSO permission set and then open Studio from the console. IAM mode is also the only mode that lets you [lock down where users can open Studio from](https://docs.aws.amazon.com/sagemaker/latest/dg/onboard-custom.html#onboard-custom-authentication-details "Authentication methods"). VPC-only mode from [Part 1](/articles/securing-sagemaker-vpc-only-mode/) keeps Studio's own traffic private, but it doesn't stop a user opening Studio from a laptop on the internet. I'll show how to restrict that below.

Here's how the two compare:

| | IAM | SSO (IAM Identity Center) |
|---|---|---|
| Who manages identities | IAM, often with your IdP federated in through SAML | IAM Identity Center, often synced from your IdP |
| How users open Studio | SageMaker AI console, or a presigned URL (`CreatePresignedDomainUrl`) | AWS access portal |
| Which user profile they land in | Any profile their sign-in role allows; restrict it with the tag policy below | The profile created for them |
| Restrict where Studio is opened from (VPC endpoint, IP range) | Yes | No |
| GovCloud (as of November 2026) | Supported | Not available |
| Change it after the domain is created | No | No |

If you choose IAM authentication, one of the _"gotchas"_ that you need to understand is that by default, SageMaker does not restrict users logging in to another user's profile, thus mimicking user impersonation.
It may be for collaboration purposes, but if users require deeper security or at least development privacy, one trick I often deploy is IAM policies with conditions that use [Global Context Keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-userid "AWS global condition context keys").

## How do you stop SageMaker Studio users opening each other's profiles?

So, remember, by default, SageMaker isn't restrictive on user profiles. Applying condition keys to `DescribeUserProfile` and `CreatePresignedDomainUrl` actions can help mitigate that collaborative openness until you, the admin, allow it.

- [Describe User Profile](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_DescribeUserProfile.html "DescribeUserProfile") - grants an IAM identity (user, group, or role) the authority to retrieve detailed metadata about a specific Amazon SageMaker Studio user profile
- [Create Presigned Domain URL](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreatePresignedDomainUrl.html "CreatePresignedDomainUrl") - grants an IAM identity (user, group, or role) the authority to generate a URL that signs into the Studio domain as a specific user profile

In your IaC repo, tag each user profile you create with a `userId` tag, and set it to the user's login email, which is typical in enterprise organizations.
Then add conditions to the policy attached to the user's sign-in role, so the API call is checked against that resource tag.

What's happening exactly is that when a user signs in to AWS, there are certain attributes associated with that user. This design grabs one of those attributes and locks down the completion of that API call, allowing or denying the API call based on the attribute you add to the policy.

For the sign-in role, identify the IAM role or SSO permission set your users generally use for AWS access, and attach a policy with the global condition keys to restrict which user profiles they can see and open.

```hcl
data "aws_partition" "current" {}

locals {
  # Partition-aware, so the same code works in commercial Regions and GovCloud
  user_profile_arn = "arn:${data.aws_partition.current.partition}:sagemaker:${var.region}:${var.account}:user-profile/${var.domain_id}/*"
}

resource "aws_iam_policy" "sagemaker_restricted_policy" {
  name        = "SageMakerSelfServiceRestrictedPolicy"
  path        = "/"
  description = "Allows profile descriptions and presigned URLs restricted by global user identity attributes"
  tags        = var.tags

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "RestrictDescribeUserProfile"
        Effect = "Allow"

        Action = [
          "sagemaker:DescribeUserProfile",
        ]

        Resource = [
          local.user_profile_arn,
        ]

        Condition = {
          StringLike = {
            "aws:userid" = [
              "*:$${sagemaker:ResourceTag/userId}"
            ]
          }
        }
      },
      {
        Sid    = "RestrictPresignedUrl"
        Effect = "Allow"

        Action = [
          "sagemaker:CreatePresignedDomainUrl"
        ]

        Resource = [
          local.user_profile_arn,
        ]

        Condition = {
          StringLike = {
            "aws:userid" = [
              "*:$${sagemaker:ResourceTag/userId}"
            ]
          }
        }
      },
      {
        # Stop users retagging a colleague's profile with their own email
        Sid    = "DenyUserProfileTagChanges"
        Effect = "Deny"

        Action = [
          "sagemaker:AddTags",
          "sagemaker:DeleteTags",
        ]

        Resource = [
          local.user_profile_arn,
        ]
      }
    ]
  })
}
```

If you mimic the above example, when your users try to open a user profile that does not match their username construction, the API call will return a _"not authorized"_ type message preventing the UI action.

A few caveats before you rely on this:

- **It only works for role sessions.** For a role session, `aws:userid` is `role-id:session-name`, and the `*:` in the condition matches everything before the session name. An IAM user's `aws:userid` is just its unique ID with no email in it, so the condition never matches IAM users.
- **The session name has to come from your IdP.** SAML sets it from the `RoleSessionName` attribute, and IAM Identity Center sets it to the user's username. I've run this pattern with SAML federation. If you're on Identity Center, run `aws sts get-caller-identity` as one of your users first, and tag profiles with whatever comes after the colon in `UserId`. If your usernames aren't email addresses, that's what goes in the tag. If users can call `sts:AssumeRole` on the sign-in role themselves, they can pick any session name, including a colleague's email. Lock that down with an `sts:RoleSessionName` condition in the role's trust policy.
- **Users can't be allowed to change the tag.** Anyone who can call `sagemaker:AddTags` on a user profile can retag it with their own email. The third statement denies that, so keep tag management in your IaC pipeline.
- **Match the whole tag.** There's no trailing `*` after the tag. With one, a profile tagged `alice@example.com` would also open for a session named `alice@example.company.com`.

That covers which profile a user can open. The next question is where they can open it from.

### How do you restrict where users can open Studio from?

Even with VPC-only mode and a Studio interface endpoint, [users outside your VPC can still connect to Studio over the internet](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-interface-endpoint.html#studio-private-link-restrict "Allow Access Only from Within Your VPC"). Any user with valid credentials can call `CreatePresignedDomainUrl` from anywhere and open the URL it returns.

To close that, add a network condition on the same two actions, `CreatePresignedDomainUrl` and `DescribeUserProfile`, in the policy on the user's sign-in role. AWS supports these condition keys:

- `aws:SourceVpce`: only through specific interface endpoints. This is the tightest option, and the one I'd pair with Part 1.
- `aws:SourceVpc`: only from inside a specific VPC.
- `aws:SourceIp`: only from public IP ranges you list, such as your office or VPN egress IPs.
- `aws:VpcSourceIp` with `aws:SourceVpc`: only from specific private IP ranges inside your VPC, such as a jump host subnet.

Here's the `aws:SourceVpce` version. It's a separate deny policy, attached to the same sign-in role, so the tag-based policy above doesn't change:

```hcl
resource "aws_iam_policy" "sagemaker_studio_network_restriction" {
  name        = "SageMakerStudioNetworkRestriction"
  path        = "/"
  description = "Denies opening Studio unless the request comes through the SageMaker API interface endpoint"
  tags        = var.tags

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyStudioAccessOutsideVpce"
        Effect = "Deny"

        Action = [
          "sagemaker:CreatePresignedDomainUrl",
          "sagemaker:DescribeUserProfile",
        ]

        Resource = [
          local.user_profile_arn,
        ]

        # Requests that don't come through this endpoint have no matching
        # aws:SourceVpce value, so they're denied
        Condition = {
          StringNotEquals = {
            "aws:SourceVpce" = var.sagemaker_api_vpce_id
          }
        }
      }
    ]
  })
}
```

Before you turn this on, know what it changes for your users:

- **They need a path into the VPC.** That means VPN, Direct Connect, or a workspace that runs inside the VPC. They also need the `api` and `studio` interface endpoints from Part 1. `CreatePresignedDomainUrl` goes through the API endpoint, and the URL it returns opens through the Studio endpoint.
- **The SageMaker AI console stops working for Studio.** The console's request doesn't come through your endpoint, so it's denied. Users have to generate the presigned URL themselves with the CLI, an SDK, or an internal portal that calls the API through the endpoint.
- **It only works in IAM-mode domains.** Identity Center mode doesn't support it.

So far everything has been about getting into Studio. Once a user is in, a different role takes over.

## What can users access once they're in Studio? The execution role

Finally, the execution role. Once a user is in Studio, SageMaker restricts or allows every action they take through their user profile's execution role, on their behalf. If a user wants to access an AWS resource, let's say EMR or S3, this role needs access granted to it.

This is where the magic of team-level actions can be addressed for important access like S3 cross-account access:

```hcl
# 1. Define the Execution Role Trust Relationship
# Only the SageMaker service can assume this role, never the user directly.
data "aws_iam_policy_document" "sagemaker_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["sagemaker.amazonaws.com"]
    }
  }
}

# 2. Create the SageMaker Execution Role
resource "aws_iam_role" "sagemaker_execution_role" {
  name               = "sagemaker-execution-role"
  assume_role_policy = data.aws_iam_policy_document.sagemaker_assume_role.json
}

# 3. Attach Cross-Account S3 Permissions to the Execution Role
# The bucket's owning account must also allow this role in its bucket policy
# (and in the KMS key policy, if the bucket uses SSE-KMS).
data "aws_iam_policy_document" "cross_account_s3" {
  statement {
    sid       = "ListTeamBucket"
    effect    = "Allow"
    actions   = ["s3:ListBucket"]
    resources = [var.team_bucket_arn]
  }

  statement {
    sid       = "ReadWriteTeamData"
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = ["${var.team_bucket_arn}/*"]
  }
}

resource "aws_iam_role_policy" "cross_account_s3" {
  name   = "cross-account-s3"
  role   = aws_iam_role.sagemaker_execution_role.id
  policy = data.aws_iam_policy_document.cross_account_s3.json
}

# 4. Assign the Execution Role directly to the User Profile
resource "aws_sagemaker_user_profile" "data_scientist_profile" {
  domain_id         = var.domain_id
  user_profile_name = "alice-data-scientist"

  user_settings {
    # This links the runtime permissions to this specific user profile
    execution_role = aws_iam_role.sagemaker_execution_role.arn
  }

  # The tag the sign-in policy above matches against aws:userid
  tags = {
    userId = "alice@example.com"
  }
}
```

Once you've gotten those roles ironed out and policies attached, you've made significant progress on team security assurance.

## Let's recap

OK, so now you've:

- Secured the network to communicate privately
- Used per-team domain deployments and execution roles
- Chosen a user authentication method for the domain
- Secured your user profiles with `userId` tags and restrictive policies
- Optionally limited where users can open Studio from with `aws:SourceVpce`

Users should now feel comfortable working in a secured environment. The next step to address is securing programmatic actions and workflows. After all, once you've finished working in a notebook, you want automated training, promotion, and endpoints.

*Previously: [Part 1 — Private networking with VPC-only mode](/articles/securing-sagemaker-vpc-only-mode/)*
