+++
title = "Handling extant resources in Terraform"
date = 2015-01-27
+++

[Terraform](https://www.terraform.io/) is a Hashicorp tool which embraces the Infrastructure as Code model to manage a variety of platforms and services in today's modern, cloud-based Internet.  It's still in development, but it already provides a wealth of useful functionality, notably with regards to Amazon and Digital Ocean interactions.  The one thing it doesn't do, however, is manage *pre-existing* infrastructure very well.  In this blog post we'll explore a way to integrate extant infra into a basic Terraform instance.
Note that this post is current as of Terraform v[0.3.6](https://github.com/hashicorp/terraform/tree/v0.3.6).  Hashicorp has hinted that future versions of Terraform will handle this problem in a more graceful way, so be sure to check those changelogs regularly. :)

# summary

A full example and walk-through will follow; however, for those familiar with Terraform and just looking for the tl;dr, I got you covered.

* Declare a *new, temporary* resource in your Terraform plan that is *nearly identical* to the extant resource.
* `Apply` the plan, thus instantiating the temporary "twinned" resource and building a state file.
* Alter the appropriate `id` fields to be the *same* as the extant resource in both the state and config files.
* Perform a `refresh` which will populate the state file with the correct data for the declared extant resource.
* Remove the temporary resource from AWS manually.
* Voilà.

## faster and more dangerous, please.

Walking through the process and meticulously checking every step? Ain't nobody got time for that!

* Edit the state file and insert the resource directly - it's just JSON, after all.

# examples

In the examples below, the notation `[...]` is used to indicate truncated output or data.
Also note that the AWS cli tool is assumed to be configured and functional.

## S3

The extant resource in this case is an S3 bucket called `phrawzty-tftest-1422290325`. This resource is *unknown* to Terraform.

```bash
$ aws s3 ls | grep tftest
2015-01-26 17:39:07 phrawzty-tftest-1422290325
```

Declare the temporary twin in the Terraform config:

```bash
resource "aws_s3_bucket" "phrawzty-tftest" {
    bucket = "phrawzty-tftest-1422353583"
}
```

Verify and prepare the plan:

```bash
$ terraform plan -out=terratest.plan
    [...]
Path: terratest.plan

+ aws_s3_bucket.phrawzty-tftest
    acl:    "" => "private"
    bucket: "" => "phrawzty-tftest-1422353583"
```

Apply the plan (this will create the twin):

```bash
$ terraform apply ./terratest.plan
    [...]
aws_s3_bucket.phrawzty-tftest: Creation complete

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
    [...]
State path: terraform.tfstate
```

Verify that the both the extant and temporary resources exist:

```bash
$ aws s3 ls | grep phrawzty-tftest
2015-01-26 17:39:07 phrawzty-tftest-1422290325
2015-01-27 11:14:09 phrawzty-tftest-1422353583
```

Verify that Terraform is aware of the temporary resource:

```bash
$ terraform show
aws_s3_bucket.phrawzty-tftest:
  id = phrawzty-tftest-1422353583
  acl = private
  bucket = phrawzty-tftest-1422353583
```

Alter the config file:

* Insert the name of the extant resource in place of the temporary.
* Strictly speaking this is not necessary, but it helps to keep things tidy.

```bash
resource "aws_s3_bucket" "phrawzty-tftest" {
    bucket = "phrawzty-tftest-1422290325"
}
```

Alter the state file:

* Insert the name (`id`) of the extant resource in place of the temporary.

```bash
            "resources": {
                "aws_s3_bucket.phrawzty-tftest": {
                    "type": "aws_s3_bucket",
                    "primary": {
                        "id": "phrawzty-tftest-1422290325",
                        "attributes": {
                            "acl": "private",
                            "bucket": "phrawzty-tftest-1422290325",
                            "id": "phrawzty-tftest-1422290325"
                        }
                    }
                }
            }
```

Refresh the Terraform state (note the ID):

```bash
$ terraform refresh
aws_s3_bucket.phrawzty-tftest: Refreshing state... (ID: phrawzty-tftest-1422290325)
```

Verify that Terraform is satisfied with the state:

```bash
terraform plan
Refreshing Terraform state prior to plan...

aws_s3_bucket.phrawzty-tftest: Refreshing state... (ID: phrawzty-tftest-1422290325)

No changes. Infrastructure is up-to-date. This means that Terraform
could not detect any differences between your configuration and
the real physical resources that exist. As a result, Terraform
doesn't need to do anything.
```

Remove the temporary resource:

```bash
$ aws s3 rb s3://phrawzty-tftest-1422353583/
remove_bucket: s3://phrawzty-tftest-1422353583/
```

## S3, faster.

For the sake of this example, the state file already contains an S3 resource called `phrawzty-tftest-blah`.
Add the "extant" resource directly to the state file.

```bash
            "resources": {
                [...]
                },
                "aws_s3_bucket.phrawzty-tftest": {
                    "type": "aws_s3_bucket",
                    "primary": {
                        "id": "phrawzty-tftest-1422290325",
                        "attributes": {
                            "acl": "private",
                            "bucket": "phrawzty-tftest-1422290325",
                            "id": "phrawzty-tftest-1422290325"
                        }
                    }
                }
```

Refresh:

```bash
$ terraform refresh
aws_s3_bucket.phrawzty-tftest: Refreshing state... (ID: phrawzty-tftest-1422290325)
aws_s3_bucket.phrawzty-tftest-blah: Refreshing state... (ID: phrawzty-tftest-blah)
```

Verify:

```bash
$ terraform show
aws_s3_bucket.phrawzty-tftest:
  id = phrawzty-tftest-1422290325
  acl = private
  bucket = phrawzty-tftest-1422290325
aws_s3_bucket.phrawzty-tftest-blah:
  id = phrawzty-tftest-blah
  acl = private
  bucket = phrawzty-tftest-blah
```

That's that.