# AWS-Config-rule-elbv2-scheme-to-tag
AWS Firewall Manager and other services can use resource tags to include or exclude resources from a policy.  A common example of this is Application Load Balancers (ALBs) and AWS WAF Firewall Manager policies to public (internet-facing) or internal only ALBs.  Customers currently need to update code to manually tag their ALBs with the scheme of the ALB.  This can be a significate effort to go back and update existing ALBs and depends on the application team specifying the accurate value.

This custom config rule automatically determines the scheme "internet-facing or internal" based on the response from DescribeLoadBalancers and ensures a tag named elbv2Scheme has the relevant value.  If the tag does not exist, the value does not match the scheme or is not one of "internet-facing" or "internal" it updates to match the value retrieved from DescribeLoadBalancers.

This solution does not prevent updating/adding the tag with a value that does not match the scheme however will automatically remediate as Config detects the resource/updates.


## Deployment
Deploy as a CloudFormation StackSet to all relevant region(s).  (Optional) Add a tag/value to the CloudFormation Stack (so that is propagates to all supported resources, specifically the IAM roles).

### Deployment Variables
> Update each variable with the desired Org Parent root and list of region(s) to deploy the stack set

```
export ParentRoot="r-1234"
export Regions="us-east-1 us-east-2 us-west-1 us-west-2"
```

### Create Stack Set
```
aws cloudformation create-stack-set \
  --stack-set-name alb-scheme-tag-aws-config-rule \
  --template-body file://alb_scheme_tag.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=true
```
### Create Stack Instances
```
aws cloudformation create-stack-instances \
--stack-set-name alb-scheme-tag-aws-config-rule \
--regions $Regions \
--deployment-targets OrganizationalUnitIds=$ParentRoot \
--operation-preferences RegionConcurrencyType=PARALLEL,MaxConcurrentPercentage=100
```

## SCP - Prevent unauthorized tag manipulation
While config will detect incorrect values and remediate, you can block the ability to add/update/remove this tag except by the remediation role.  Below is a SCP statement which denies  IAM principals without the tag pair, RoleType: RestrictedConfigRemediationRole

> Ensure users cannot bypass this by creating their own roles with this tag too.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Statement1",
      "Effect": "Deny",
      "Action": [
        "elasticloadbalancing:RemoveTags",
        "elasticloadbalancing:AddTags"
      ],
      "Resource": [
        "*"
      ],
      "Condition": {
        "StringLike": {
          "aws:RequestTag/elbv2Scheme": [
            "*"
          ]
        },
        "StringNotEquals": {
          "aws:PrincipalTag/RoleType": [
            "RestrictedConfigRemediationRole"
          ]
        }
      }
    }
  ]
}
