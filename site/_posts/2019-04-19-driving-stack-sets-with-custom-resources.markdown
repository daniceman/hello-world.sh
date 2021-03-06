---
layout: post
title: "Driving CloudFormation Stack Sets with Custom Resources"
date: 2019-04-19
---

This post will discuss the usage of CloudFormation Custom Resources as a
frontend to CloudFormation Stack Sets so you can use a CloudFormation template
to use CloudFormation Stack Sets to deploy lots of CloudFormation stacks. Wait,
**WHAT?!**

![Yo dawg, I heard you like CloudFormation, so I put some CloudFormation on your CloudFormation](/assets/images/driving-stack-sets-with-custom-resources/xzibit.jpg)

# CloudFormation Stack Sets

CloudFormation Stack Sets is a great way to deploy uniform configurations of
resources into multiple regions and accounts in your AWS Organization. It's a
great feature for large scale deployments, really. It is particularly useful in
deploying compliance mechanisms as part of landing zone implementations. A good
example would be a deployment of AWS Config rules to track whether all RDS
database instances and/or snapshots are encrypted across every region and
across every account in your AWS Organization.

Stack Sets allow you to define a uniform CloudFormation template and create a
so-called stack set definition around it. This definition holds configuration
parameters as to how to deploy the associated CloudFormation template. It
includes role definitions for CloudFormation to actually deploy things as well
as other various parameters to define how the deployment happens. Once the
definition is created, it is possible to create so-called stack instances in
the set. An instance is basically a representation of a CloudFormation stack
deployed into a single account into a single region. Afterward, CloudFormation
Stack Sets will then attempt to actually deploy the stack as part of an
operation.

However, CloudFormation Stack Sets do not come with actual CloudFormation
support themselves. Other CloudFormation resource types like Macros or Custom
Resources actually are supported and you are able to create their definitions
in native CloudFormation templates. Stack Sets, however, lack this support.

Luckily, the world has [Chuck Meyer](https://twitter.com/chuckm) in it and
Chuck wrote a CloudFormation Custom Resource to drive CloudFormation Stack Set
deployments. The code for the Custom Resource is available [on
Github](https://github.com/awslabs/aws-cloudformation-templates/tree/master/aws/solutions/StackSetsResource).

# Deploying the Custom Resource

Deployment of the Custom Resource is very straight forward and works exactly as
Chuck describes it in his
[README.md](https://github.com/awslabs/aws-cloudformation-templates/blob/master/aws/solutions/StackSetsResource/README.md).
The Custom Resource gets packaged and deployed with CloudFormation (of course).
Afterward, the Stack Set resource type will be available as `Custom::StackSet`.

To effectively execute Stack Set deployments driven by the resource, we need to
also provision some IAM roles, so that CloudFormation can
actually deploy stacks on you behalf. Two roles are needed:

First off, CloudFormations service principal `cloudformation.amazonaws.com`
needs to be able to `sts:AssumeRole`. This role needs to be present in the
administrative AWS account of the landing zone (I am assuming you know what a
landing zone is) and from now on, we will refer to this role as the
'AdministrationRole'.

Furthermore, we need to provision the 'ExecutionRole'. The ExecutionRole needs
to be available in the accounts in which we actually want to deploy resources
through Stack Sets. This role needs to have a trust relationship with your
administrative AWS account or the AdministrationRole specifically. It also
needs to have enough permissions to execute the CloudFormation template that
you would like to drive through Stack Sets. It might be a good idea to re-use
the administrative role that may or may not be part of your landing zone
environement.

Once all of these roles are in place, a typical Stack Set driven deployment of
a template would assume the roles as follows:

```plain
              sts:AssumeRole                sts:AssumeRole          deployment

CloudFormation            AdministrationRole           ExecutionRole
                +------>                      +------>                 +--->  StackInstance
administrative              administrative    |           target
   account                     account        |          account A
                                              |
                                              |
                                              |
                                              |        ExecutionRole
                                              +------>                 +--->  StackInstance
                                                          target
                                                         account B
```

After putting everything in place, we finally get to actually use the new
abstraction layer we build. The Custom Resource will allow us to use
CloudFormation Stack Sets through the prefered interface: CloudFormation.

# Creating the 'Super Template'

Finally, it's time to look at the 'outer' CloudFormation template. This
template will instantiate the new Custom Resource, which in turn will create,
update or delete Stack Set instances that actually run the 'inner'
CloudFormation template.

For this, the inner CloudFormation template needs to be available on S3. I
recommend appending a hash value of the template contents (or a version
identifier) to its S3 object key, so that the outer template can be informed of
a change in the inner template.

The `.yaml` below is an example of the outer template. 

```yaml
Parameters:
  AdministratonRole:
    Type: String
    Description: ARN of the AdministrationRole (local switch-role).
  ExecutionRole:
    Type: String
    Description: Name of the ExecutionRole (Role to assume in target accounts).
  TemplateURL:
    Type: String
    Description: S3 URL of template to deploy to target accounts.
    AllowedPattern: ^https://s3(.+)\.amazonaws.com/.+$
  SetName:
    Type: String
    Description: Name of this Stack Set
  Accounts:
    Type: List<String>
    Description: List of target accounts
  Regions:
    Type: List<String>
    Description: List of target regions

Resources:
  StackSet:
    Type: Custom::StackSet
    Properties:
      ServiceToken:
        Fn::ImportValue: StackSetCustomResource
      StackSetName: !Ref SetName
      TemplateURL: !Ref TemplateURL
      Capabilities:
        - CAPABILITY_IAM
      AdministrationRoleARN: !Ref AdministrationRole
      ExecutionRoleName: !Ref ExecutionRole
      OperationPreferences: {
        "FailureToleranceCount": 500,
        "MaxConcurrentCount": 500
      }
      Tags:
        - Creator: Daniel Stamer
        - Mail: dan@hello-world.sh
      StackInstances:
        - Regions: !Ref Regions
          Accounts: !Ref Accounts
```

You can execute this template directly through the common CloudFormation
interfaces such as API, CLI or the Web Console. Alternatively, you can use an
automation driver tool like [Sceptre](https://github.com/cloudreach/sceptre) to
automate the upload of the inner template or to dynamically retrieve
configuration for target accounts and regions.

If you want to stay _completely native_ to CloudFormation, you can look at
[CloudFormation Macros](/2018/10/19/cloudformation-macros.html) to retrieve
dynamic parameter configuration through a Lambda-based backend.

# Limits of CloudFormation Stack Sets

There are two soft limits of Stack Sets that will affect your endeavors
especially when you are operating are large scales. First of all, you can only
have 20 Stack Set definitions. This is not so bad, but it might affect the way
you are writing your inner templates. If you are used to having very few
resources in your template, you will run into this limit quickly. Grouping
resources into larger purpose-bound templates certainly helps. The second soft
limit is more severe. Per default, a single Stack Set definition can only
contain up to 500 stack instances. 500 seems like a lot, but if you are running
a platform with 100 AWS accounts and you want to deploy i.e. AWS Config recorders
and rules to more than 5 regions, then you are having a problem. Note that this
is an absolutely valid use case. Now, AWS might increase that soft limit for
you if you manage to convince them of the validity of your use case. Grouping
these kinds of deployments into separate Stack Set definitions might also be an
alternative, but then you'd be affected by the first soft limit sooner.

# Conclusion

The interesting feature of Stack Sets is the ability to not just execute a
CloudFormation template in the local account and region, but to bulk execute
them over a large set of accounts & regions. Now, the standard Stack Set
interface itself is not very CloudFormation-ish in the sense that you are not
expressing a declarative desired state. The Custom Resource for Stack Sets
makes Stack Sets act almost exactly like other resources managed by
CloudFormation. The user can express a desired state and the service takes care
of drifting the actual state towards the expressed desired one.

The implications for platform automation tools are potentially huge. If you are
a platform operator and you are maintaining systems like an Account Vending
Machine or a Landing Zone bootstrapping tool, then you can already see the
possible applications for Stack Sets. Platform tooling, which continously
re-runs tasks to ensure the presence of i.e. compliance configurations over
hundreds of AWS accounts, can massively benefit from Stack Sets.

