+++
title = "Structuring Terraform Projects"
date = "2019-04-14"
author = "Pierce Bartine"
description = "A flexibly structured Terraform project enables developer productivity"
tags = ["software", "devops", "terraform"]
lastmod = "2019-04-14"
slug = "structuring-terraform-projects"
+++

> _Measure twice, cut once_

Quick intro: Terraform is a tool made by Hashicorp to enable tracking the state of your infrastructure. You declare the desired state of your infrastructure in code using HCL, then pass it through the Terraform binary, which makes it so.

There are a few phases teams go through when adopting Terraform, and we're going to take a look at the workflow and challenges associated with each step of the process.

### Phase 0: Someone catches the bug

- Number of users: 1
- Number of repositories: 0-1
- State: local
- Modules: none

One of the devs, Armon, found out about Terraform, and plays around with it on his local machine. He spins up a few EC2 instances as a test. Since Armon is the only user, he doesn't bother using a remote state backend, and probably doesn't commit anything to a Git repo. After all, this is just a test.

### Phase 1: The bug starts the spread (Forming)

- Number of users: 2-5
- Number of repositories: 1-2
- State: remote, monolithic
- Modules: none

Armon ends up showing Terraform to his team and they like it enough to start using it. The team creates a repository or two to track their code. Most of their AWS infrastructure is still created manually or with some other tool, to the chagrin of the newfound Hashifans.

The team quickly realizes they needed to store their Terraform state remotely when one of the devs, Mitchell, checks out the code to his machine and runs a `terraform plan` that attempts to create all new infrastructure. He discusses this with Armon, who previously had been running all the plans and applies from his machine. They come to the conclusion to use Terraform's S3 [remote state backend][1] (with DynamoDB for locking). This way, Armon, Mitchell, and the rest of the team members can run Terraform commands on their own machines, while the authoritative version of the state lives remotely - not on one of the dev's machine.

The team's codebase right now consists of a Git repo with a bunch of `.tf` files in its root. Somewhere in one of those files is the configuration for the remote state backend:

```json
terraform {
  backend "s3" {
    # ...
```

### Phase 2: Massive proliferation (still Forming)

- Number of users: 5-10
- Number of repositories: 6-10
- State: remote, distributed
- Modules: one or two, local only

The team's initial experience with Terraform has been fantastic. They're getting used to its quirks, have bookmarked the [interpolation syntax][2] page, and have adopted it as their infrastructure-as-code tool of choice. They're now using it to track all of their AWS infrastructure across all of their environments: dev, staging, and prod.

Armon's specialty on the team is networking, so he creates a repository for his networking infrastructure, for each environment, like so: `net-dev`, `net-staging`, `net-prod`. Seeing this, Mitchell, the security expert, follows suit and creats an `iam-` Terraform repository for each environment. Finally, Jannet, the lead of the app team, creates repos for her team's app servers, with the same naming convention `app-`.

By now there are a lot of team members using Terraform on a daily basis. The team discovers Atlantis, an open source project that allows them to manage the `terraform plan && apply` lifecycle from GitHub PRs. It makes work visible, and since all of the applies are serialized through the Atlantis server, developers can be sure that whatever infra is declared in the Git repo's master branch is what exists in AWS.

### Phase 3: The bug is making people sick (Storming)

- Number of users: 10+
- Number of repositories: 10+
- State: remote, distributed
- Modules: mixture of local and remote

Terraform adoption has spread like wildfire, and almost every developer in the company has heard of it in some fashion now. They're all trying to use it now, but Silicon Valley's age-old problem arises: it doesn't scale. 

Because there are so many Terraform repositories, devs frequently find themselves having to make changes to two or more of them to affect one atomic change. For example:

Mitchell wants to deploy another one of Hashicorp's tools, Vault. He's caught at a crossroad though: Vault requires IAM resources, which would naturally belong in the `iam-` repos, and likewise for the networking and EC2 pieces. Not to mention bubbling the up from dev to staging to prod requires a copy-paste per hop!

Additionally, developers new to Terraform want to be able to iterate faster when creating their infrastructure definitions, but since they don't have access to run Terraform on their local machines, they iterate through Atlantis using long-lived PRs. Due to Atlantis's lock, these PRs frequently end up blocking each other as the devs go through the motions of `git commit -am 'test' && git push` and then `atlantis plan -> bug team for approval -> atlantis apply`.

### Phase 4: Things are looking up (Norming)

- Number of users: 10+
- Number of repositories: ~3 for instantiation, many for modules
- State: remote, distributed
- Modules: mostly remote

The team begins to use remote Terraform modules extensively, gaining the ability to version modules and bubble changes up through the environments gradually.

{{< figure src="/images/tfstatestructure.png"
  alt="Terraform State Structure"
  position="center"
  style="border-radius: 8px;"
  captionPosition="center" >}}


[1]: https://www.terraform.io/docs/backends/index.html
[2]: https://www.terraform.io/docs/configuration-0-11/interpolation.html
