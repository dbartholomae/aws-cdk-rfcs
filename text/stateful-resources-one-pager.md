# 1Pager | Modeling Stateful Resources

This document is a one pager meant to surface the idea of explicitly modeling stateful resources in the core framework. Its goal is to conduct a preliminary discussion and gather feedback before it potentially progresses into a full blown RFC.

## Customer pain

Customers are currently exposed to un intentional data loss when stateful resources are designated for either removal or replacement by CloudFormation.

Data, similarly to security postures, is an area in which even a single rare mistake can cause catastrophic outages. The CDK can and should help reduce the risk of such failures.
With respect to security, the CDK currently defaults to blocking deployments that contain changes in security postures, requiring a user confirmation:

```console
This deployment will make potentially sensitive changes according to your current security approval level (--require-approval broadening).
Please confirm you intend to make the following modifications:
...
...
Do you wish to deploy these changes (y/n)?
```

However, no such mechanism exists for changes that might result in data loss, i.e removal or replacement of stateful resources, such as `S3` buckets, `DynamoDB` tables, etc...

## Failure scenarios

To understand how susceptible customers are to this, we outline a few scenarios where such data loss can occur.

### Stateful resource without `DeletionPolicy/UpdateReplacePolicy`

By default, CloudFormation will **delete** resources that are removed from the stack, or when a property that requires replacement is changed.

> - [UpdateReplacePolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatereplacepolicy.html)
> - [DeletionPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html)

To retain stateful resources, authors must remember to configure those policies with the `Retain` value. Having to remember this makes it also easy to forget. This means that stateful resources might be shipped with the incorrect policy.

### Policy Change

CDK applications are often comprised out of many third-party resources. Even if a third-party resource is initially shipped with the correct policy, this may change. Whether or not the policy change was intentional is somewhat irrelevant, it can still be undesired and have dire implications on the consuming application.

For that matter, even policy changes made by the application author itself can be unexpected or undesired.

## Desired Customer Experience

The experience described here lays under the following assumptions:

1. The correct policy for stateful resources is always `Retain`.
2. With data loss, its better to ask for permission than ask for forgiveness.

As mentioned before, we should provide an experience similar to the one we have with respect to security posture changes. When deploying or destroying an application causes the removal of stateful resources, the user will, by default, see:

```console
The following stateful resources will be removed:

 - MyBucket (Removed from stack)
 - MyDynamoTable (Changing property `TableName` requires a replacement)

Do you wish to continue (y/n)?
```

> Note that we want to warn the users about possible replacements as well, not just removals.

If this is desired, the user will confirm. And if this behavior is expected to repeat, the user can run:

`cdk deploy --allow-removal MyBucket --allow-removal MyDynamoTable`

To skip the interactive confirmation.

An additional feature we can provide is to detect when a deployment will change the policy of stateful resources. i.e:

```console
The removal policy of the following stateful resources will change:

 - MyBucket (RETAIN -> DESTROY)
 - MyDynamoTable (RETAIN -> DESTROY)

Do you wish to continue (y/n)?
```

Blocking these types of changes will keep the CloudFormation template in a safe state.
Otherwise, if such changes are allowed to be deployed, out of band operations
via CloudFormation will still pose a risk.

## High Level Design

In order to provide any of the features described above, the core framework must be able
to differentiate between stateful and stateless resources.

> Side note: Explicitly differentiating stateful resources from stateless ones is a common practice in infrastructure modeling. For example, Kubernetes uses the [`StatefulSet`](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) resource for this. And terraform has the [`prevent_destroy`](https://www.terraform.io/language/meta-arguments/lifecycle#prevent_destroy) property to mark stateful resources.

### API

We'd like to extend our API so it enforces every resource marks itself as either stateful or stateless.

To that end, we propose adding the following properties to the `CfnResourceProps` interface:

```ts
export interface CfnResourceProps {
  /**
   * Whether or not this resource is stateful.
   */
  readonly stateful: boolean;

  /**
   * Which properties cause replacement of this resource.
   * Required if the resource is stateful, encouraged otherwise.
   *
   * @default undefined
   */
  readonly replacedIf?: string[];
}
```

### CodeGen

Adding those properties will break the generated L1 resources because the `stateful` property is required.
This means we need to add some logic into `cfn2ts` so that it generates constructors that pass the appropriate values.

To support this, we manually curate a list of stateful resources and check that into our source code, i.e:

+ stateful-resources.json

```json
{
  "AWS::S3::Bucket": ["BucketName"],
  "AWS::DynamoDB::Table": ["TableName"],
}
```

The key is the resource type, and the value is the list of properties that require replacement.

> We might be able to leverage the work already done by cfn-lint: [StatefulResources.json](https://github.com/aws-cloudformation/cfn-lint/blob/main/src/cfnlint/data/AdditionalSpecs/StatefulResources.json)
> However, we also need the list of properties that require replacement for these resources. We can probably collaborate with cfn-lint on this.

Using this file, we can generate the correct constructor, for example:

```ts
export class CfnBucket extends cdk.CfnResource implements cdk.IInspectable {

  constructor(scope: cdk.Construct, id: string, props: CfnBucketProps = {}) {
      super(scope, id, {
        stateful: true,
        replacedIf: ["BucketName"],
        type: CfnBucket.CFN_RESOURCE_TYPE_NAME,
        properties: props
      });
      ...
      ...
  }
}
```

### Synthesis

### Deployment





## Q & A

#### Isn't remembering to extend `StatefulResource` the same as remembering to configure `DeletionPolicy/UpdateReplacePolicy`?

#### What about CDK Pipelines?

#### Would this have prevented https://github.com/aws/aws-cdk/issues/16603?

####
