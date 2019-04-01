# AWS IAM

## Contents

- [Resources](#Resources)
- [Summary](#Summary)
- [Other layers of access control on AWS](
  #Other-layers-of-access-control-on-AWS)
- [Further network security measures](#Further-network-security-measures)
- [Examples](#Examples)
    - Locking down access to a folder on S3.

## Resources

- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Security Best Practices](https://d1.awsstatic.com/whitepapers/Security/AWS_Security_Best_Practices.pdf)
- Sources of industry-accepted system hardening standards:
    - Center for Internet Security (CIS)
    - International Organization for Standardization (ISO)
    - SysAdmin Audit Network Security (SANS) Institute
    - National Institute of Standards Technology (NIST)
- [OSSEC](https://www.ossec.net
  ) is a free, open-source host-based intrusion detection (HIDS) system.

## Summary

You create IAM users under your AWS account and then assign them permissions
directly, or assign them to groups to which you assign permissions. IAM users
can be a person, service, or application that needs access to your AWS resources
through the management console, CLI, or directly via APIs.

You can create finegrained permissions to resources under your AWS account,
apply them to groups you create, and then assign users to those groups. This
best practice helps ensure users have least privilege to accomplish tasks.

You can provide each IAM group with permissions to access AWS resources by
assigning one or more IAM policies. All policies assigned to an IAM group are
inherited by the IAM users who are members of the group.

[AWS Managed Policies for Job Functions](
  https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html)
  might save some time when writing up these policies.

### Delegation Using IAM Roles and Temporary Security Credentials

Use cases:

- Applications running on Amazon

  Applications that run on an Amazon EC2 instance and that need
  access to AWS resources such as Amazon S3 buckets or an
  Amazon DynamoDB table must have security credentials in order
  to make programmatic requests to AWS. Developers might
  distribute their credentials to each instance and applications can
  then use those credentials to access resources, but distributing
  long-term credentials to each instance is challenging to manage
  and a potential security risk.

- EC2 instances that need to access AWS resources

  Cross account access To manage access to resources, you might have multiple AWS
  accounts—for example, to isolate a development environment from
  a production environment. However, users from one account might
  need to access resources in the other account, such as promoting
  an update from the development environment to the production
  environment. Although users who work in both accounts could
  have a separate identity in each account, managing credentials for
  multiple accounts makes identity management difficult.

- Identity federation

  Users might already have identities outside of AWS, such as in your
  corporate directory. However, those users might need to work with
  AWS resources (or work with applications that access those
  resources). If so, these users also need AWS security credentials
  in order to make requests to AWS.

IAM roles and temporary security credentials address these use cases. An IAM
role lets you define a set of permissions to access the resources that a user or
service needs, but the permissions are not attached to a specific IAM user or
group.

Assuming the role returns temporary security credentials that the user or
application can use to make for programmatic requests to AWS. These temporary
security credentials have a configurable expiration and are automatically
rotated. Using IAM roles and temporary security credentials means you don't
always have to manage longterm credentials and IAM users for each entity that
requires access to a resource.

## Other layers of access control on AWS

On AWS, you can build network segments using the following access control
methods:

- Using Amazon VPC to define an isolated network for each workload or
organizational entity.

- Using security groups to manage access to instances that have similar
functions and security requirements; security groups are stateful firewalls
that enable firewall rules in both directions for every allowed and
established TCP session or UDP communications channel.

- Using Network Access Control Lists (NACLs) that allow stateless
management of IP traffic. NACLs are agnostic of TCP and UDP sessions,
but they allow granular control over IP protocols (for example GRE, IPSec
ESP, ICMP), as well as control on a per-source/destination IP address and
port for TCP and UDP. NACLs work in conjunction with security groups,
and can allow or deny traffic even before it reaches the security group.

- Using host-based firewalls to control access to each instance.

- Creating a threat protection layer in traffic flow and enforcing all
traffic to traverse the zone.

- Applying access control at other layers (e.g. applications and
services).

## Further network security measures

If you’re looking for defense in-depth, you should deploy a network level
security control appliance, and you should do so inline, where traffic is
intercepted and analyzed prior to being forwarded to its final destination,
such as an application server.

Examples of inline threat protection technologies include the following:

- Third-party firewall devices installed onEC2 instances (aka soft blades)
- Unified threat management (UTM) gateways
- Intrusion prevention systems
- Data loss management gateways
- Anomaly detection gateways
- Advanced persistent threat detection gateways

Latency, complexity, and other architectural constraints sometimes rule out
implementing an inline threat management layer, in which case you can choose
one of the following alternatives.

- A distributed threat protection solution: This approach installs
threat protection agents on individual instances in the cloud. A central
threat management server communicates with all host-based threat
management agents for log collection, analysis, correlation, and active
threat response purposes.
- An overlay network threat protection solution: Build an overlay
network on top of your Amazon VPC using technologies such as GRE
tunnels, vtun interfaces, or by forwarding traffic on another ENI to a
centralized network traffic analysis and intrusion detection system, which
can provide active or passive threat response.

## Examples

### Locking down access to a folder on S3

```yaml

SupportPolicy:
  Description: Allow the support group full access to the testing folder.
  Type: AWS::IAM::Policy
  Properties:
    PolicyName: SupportPolicy
    PolicyDocument:
      Statement:
      # Allow the user to see the entire bucket list. Deny this for now.
      # Note that without these permissions the user will not be able to use
      # the AWS S3 console. However, this can gain access using S3
      # Browser and CyberDuck. Using these apps the user can easily create
      # bookmarks directly to the testing folder e.g.:
      # /mybucket/testing
      # In S3 Browser a bookmark is called an "external bucket".
      - Effect: Deny
        Action:
        - s3:ListAllMyBuckets
        - s3:GetBucketLocation
        Resource:
        - arn:aws:s3:::*

      # Allow listing of the testing folder and subfolders.
      - Effect: Allow
        Action:
        - s3:ListBucket
        Resource:
        - arn:aws:s3:::mybucket
        Condition:
          StringEquals:
            s3:prefix: ["testing/"]
            s3:delimiter: ["/"]
      - Effect: Allow
        Action:
        - s3:ListBucket
        Resource:
        - arn:aws:s3:::mybucket
        Condition:
          StringLike:
            s3:prefix: ["testing/*"]

      # Allow all S3 actions in the testing folder and subfolders.
      - Effect: Allow
        Action:
        - s3:*
        Resource:
        - arn:aws:s3:::mybucket/testing
        - arn:aws:s3:::mybucket/testing/*

    Groups:
    - !Ref SupportGroup


```