+++
title = "AWS CloudWatch"
layout = "scaler"
availability = "v1.0+"
maintainer = "Community"
description = "Scale applications based on AWS CloudWatch."
go_file = "aws_cloudwatch_scaler"
+++

### Trigger Specification

This specification describes the `aws-cloudwatch` trigger that scales based on a AWS CloudWatch.

```yaml
triggers:
- type: aws-cloudwatch
  metadata:
    # Required: namespace
    namespace: AWS/SQS
    # Required: Dimension Name - Supports specifying multiple dimension names by using ";" as a separator i.e. dimensionName: QueueName;QueueName
    dimensionName: QueueName
    # Required: Dimension Value - Supports specifying multiple dimension values by using ";" as a separator i.e. dimensionValue: queue1;queue2
    dimensionValue: keda
    metricName: ApproximateNumberOfMessagesVisible
    targetMetricValue: "2"
    minMetricValue: "0"
    # Required: region
    awsRegion: "eu-west-1"
    # Optional: AWS Access Key ID, can use TriggerAuthentication as well
    awsAccessKeyIDFromEnv: AWS_ACCESS_KEY_ID # default AWS_ACCESS_KEY_ID
    # Optional: AWS Secret Access Key, can use TriggerAuthentication as well
    awsSecretAccessKeyFromEnv: AWS_SECRET_ACCESS_KEY # default AWS_SECRET_ACCESS_KEY
    identityOwner: pod | operator # Optional. Default: pod
    # Optional: Default Metrict Collection Time
    defaultMetricCollectionTime: 300 # default 300
    # Optional: Default Metric Statistic
    defaultMetricStat: "Average" # default "Average"
    # Optional: Default Metric Statistic Period
    defaultMetricStatPeriod: 300 # default 300
```

**Parameter list:**

- `identityOwner` - Receive permissions on the CloudWatch via Pod Identity or from the KEDA operator itself (see below). (Values: `pod`, `operator`, Default: `pod`, Optional)

> When `identityOwner` set to `operator` - the only requirement is that the KEDA operator has the correct IAM permissions on the CloudWatch. Additional Authentication Parameters are not required.

- `defaultMetricCollectionTime` - How long in the past (seconds) should the scaler check AWS Cloudwatch. Used to define **StartTime** ([official documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_GetMetricStatistics.html)). (Default: `300`, Optional)
`defaultMetricStat` - Which statistics metric is going to be used by the query. Used to define **Stat** ([official documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_GetMetricStatistics.html)). (Default: `Average`, Optional)
- `defaultMetricStatPeriod` - Which frequency is going to be used by the related query. Used to define **Period** ([official documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_GetMetricStatistics.html)). (Default: `300`, Optional)


### Authentication Parameters

> These parameters are relevant only when `identityOwner` is set to `pod`.

You can use `TriggerAuthentication` CRD to configure the authenticate by providing either a role ARN or a set of IAM credentials.

**Pod identity based authentication:**

- `podIdentity.provider` - Needs to be set to either `aws-kiam` or `aws-eks` on the `TriggerAuthentication` and the pod/service account must be configured correctly for your pod identity provider.

**Role based authentication:**

- `awsRoleArn` - Amazon Resource Names (ARNs) uniquely identify AWS resource.

**Credential based authentication:**

- `awsAccessKeyID` - Id of the user.
- `awsSecretAccessKey` - Access key for the user to authenticate with.

The user will need access to read data from AWS CloudWatch.

### Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secrets
data:
  AWS_ACCESS_KEY_ID: <encoded-user-id>
  AWS_SECRET_ACCESS_KEY: <encoded-key>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-aws-credentials
  namespace: keda-test
spec:
  secretTargetRef:
  - parameter: awsAccessKeyID     # Required.
    name: keda-aws-secrets        # Required.
    key: AWS_ACCESS_KEY_ID        # Required.
  - parameter: awsSecretAccessKey # Required.
    name: keda-aws-secrets        # Required.
    key: AWS_SECRET_ACCESS_KEY    # Required.
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-cloudwatch-queue-scaledobject
  namespace: keda-test
spec:
  scaleTargetRef:
    name: nginx-deployment
  triggers:
  - type: aws-cloudwatch
    metadata:
      namespace: AWS/SQS
      dimensionName: QueueName
      dimensionValue: keda
      metricName: ApproximateNumberOfMessagesVisible
      targetMetricValue: "2"
      minMetricValue: "0"
      awsRegion: "eu-west-1"
    authenticationRef:
      name: keda-trigger-auth-aws-credentials
```
