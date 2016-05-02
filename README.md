![tiller](http://i.imgur.com/TB3W2EU.jpg)

# Synopsis
`tiller` is a logical progression from `poutine` - it's a python wrapper for packer and terraform which extends `poutine` to allow for resource dependencies, passing inputs to a dependant resource from the outputs of its dependencies, and extensibility by anyone who can write packer or terraform files.

The initial reason for its creation was to allow for easy deployment of staging and development environments to match production by any engineer into his or her own AWS account.

## Contents
| Directory | Description |
| --- | --- |
| `resources` | Contains tiller _resources_. See README.md in that directory for more information on how to write a resource. |
| `tillerlib` | tiller support files | 
| `tiller.py` | The application itself | 
| `poutine` | a wrapper for terraform. Still works - use it until tiller is done. See README.old.md for its proper use. |

# Requirements

To run these tools you will need a few things:
### Installed software:
- [Terraform](https://www.terraform.io/downloads.html), if you want to build cloud infrastructure
- [Packer](https://www.packer.io/downloads.html), if you want to build images
- [direnv](http://direnv.net/) (Optional, but helpful): Allows you to export/unset/edit environment variables per directory. Create a `.envrc` file with the envvars below, and it will only have an effect when you are in the project directory after running `direnv allow`

### An (_emptyish..._) AWS account
Although **tiller** only interacts with your account when it can create or find valid state files, you should not use it with an account that contains anything you would be devastated to see accidentally destroyed until it's out of alpha.

- This tool may be create tons of resources, especially if you are building the `terraform/awsbase` resource for the first time. Chances are, unless you have
  previously taken steps to avoid it, you will hit resource limits by running
  this against an account that is not already empty.
- Set up a non-root user with administrative privileges. Note the access key ID
  and secret access key.
- Set up a role which is connected to the policy `AmazonEC2ContainerServiceforEC2Role` if you plan on using ECS and Docker.

### A bucket for secrets. A "secret bucket".
Create an S3 bucket (with versioning, preferably) to store secrets in. 

These secrets will include: 
- the active tfstate, to maintain state between terraform runs by different people
- a backup of tfstate, tagged with the user who created it, environment, and
  date. This is nice to have if the world explodes and the aforementioned tfstate can't
  be trusted to be the truth anymore.
- Chef's validation.pem file

_**NB:** Because of the sensitive nature of this data, access to this bucket should be
tighly controlled._ Open it for use to the user you created for this tool.

### A good sense of humor
Because it probably won't work the first time.

# Use
Clone the `tiller` repo and checkout `develop`:
```
$ git clone git@github.com:blueapron/tiller.git
$ git checkout develop
```
You also need `docopt` until I package everything up properly:
`$ pip install docopt`

Familiarize yourself with the CLI:
`./tiller.py --help`

Run `./tiller.py list` to see what resources are available. The items that print out will be in `<namespace>/<name>` format. You can then `./tiller.py describe <resource>` (where `<resource>` is the fully-qualified namespace/name) to see what it does, and what its dependencies are.

There is still [too much] manual setup that needs to be done, but this will be solved in future versions. For now, navigate to `resources/<namespace>/<name>` and configure the resource. If there is a `.sample` file, copy it and remove the `.sample` extension, then edit it. Anywhere you see VPC IDs, subnets, amis, etc., you will need to verify that these are correct before continuing.

You will need to set some environment variables or pass in `--var="key=value"` pairs at the command line. Get started with the ones below. You don't need all of them for all things, but until the templating system is worked out I don't really have a great list of which ones you need for any particular run. _I'm working on it..._

Consider doing this with direnv (see Requirements section).
```
export AWS_ACCESS_KEY_ID="<your key id>"
export AWS_SECRET_ACCESS_KEY="<your secret access key>"
export AWS_REGION=us-east-1
export TILLER_SECRET_BUCKET="<your secret bucket>"
export PACKER_ACCESS_KEY="${AWS_ACCESS_KEY_ID}"
export PACKER_SECRET_KEY="${AWS_SECRET_ACCESS_KEY}"
export PACKER_BUCKET_ACCESS_KEY="${AWS_ACCESS_KEY_ID}"
export PACKER_BUCKET_SECRET_KEY="${AWS_SECRET_ACCESS_KEY}"
export PACKER_REGION="${AWS_REGION}"
export AWS_DEFAULT_REGION="${AWS_REGION}"
export PACKER_VPC_ID="<your vpc>"
export PACKER_SUBNET_ID="<your subnet>"
export PACKER_SECRET_BUCKET="${TILLER_SECRET_BUCKET}"
export IAM_INSTANCE_PROFILE="AmazonECSContainerInstanceRole"
```

### See what resources are available:
`tiller.py list`

### To describe what a resource is and what its depencendies are:
`tiller.py describe <resource>`

### To plan a build:
`tiller.py plan <resource> --env=<environment>`

### To build a resource:
`tiller.py build <resource> --env=<environment>`

### Note: many of the options require an `--env` flag to work for safety. 

### Variable resolution order:
When certain variables are required, they will be resolved in this order:

For a variable named `varname`:
- Local configurations:
  - for terraform resources, `resources/terraform/name/terraform.tfvars`
  - for packer resources, `resources/packer/<build file>.json[.tiller]`
- Environment variables:
  - `TILLER_varname=value`
- Command line:
  - `--var="varname=value"`
- Any variables which are required but not otherwise set will prompt the user at runtime

## Limitations:
I am not entirely sure that `destroy` works perfectly yet, but it does (apparently) work. 


# Contributing
Please use the `develop` branch for all contributions. All changes should be made in `feature/<feature_name>` or `hotfix/<hotfix_name>` branches. For those of you using `arcanist` (_PLEASE DO_), `arc diff` and `arc push` will automatically reference `origin/develop`. Otherwise, please create your pull requests on `develop` ___and not `master`___


## TODO (21)
1. poutine/tiller.py:76          Rather than compile all resources and pick one, start by assuming we 
2. poutine/tiller.py:148         update check_deps to wrap a build/plan/etc function to check deps
3. poutine/tiller.py:201         Finish 'plan'
4. poutine/tiller.py:203         the following pattern is used multiple times.
5. poutine/tiller.py:219         finish 'build'
6. poutine/tiller.py:236         Implement 'show'
7. poutine/tiller.py:250         Implement 'destroy'
8. discrete-service/main.tf:27   should move this to a module, or something similar and share the rules
9. discrete-service/main.tf:33   move to only whitelist office ip?
10. docker-ecs/main.tf:19        Investigate what SG rules are needed for apps to work
11. docker-ecs/main.tf:85        Figure out subnets automatically. Either use a module, or pre-fill from network output
12. tillerlib/tillerlib.py:75    Implement get_var
13. resources/packer.py:43       Parse packer vars first, whatever that actually means
14. resources/packer.py:163      change this so we can figure out what variables to require.
15. resources/packer.py:223      implement PackerResource.plan()
16. resources/packer.py:228      packer inspect packerfile.json
17. resources/terraform.py:36    add JSON parsing to this list.
18. resources/terraform.py:89    if config.tiller has a state_key_ext or state_key_var field, add
19. resources/terraform.py:187   Better buckets - use config hierarchy (implement tl.getvar())
- [x] resources/terraform.py:301   implement TerraformResource.show()
21. resources/terraform.py:312   Handle force flag correctly.
22. Explain how to set up resources by hand.
23. Make it so you don't gotta do that^^
24. Generate keypairs
25. Configurable ami in base packer image. Ideally, find this based on region