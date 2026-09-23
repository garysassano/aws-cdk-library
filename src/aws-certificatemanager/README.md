# ACM certificates

`DnsValidatedCertificateV2` creates a public DNS-validated ACM certificate with native CloudFormation resources. Set `certificateRegion` to create the certificate in another region without defining an owner stack. The construct re-imports the Route 53 zone into that owner and checks certificate names against zone authority.

Core CDK can already share a `Certificate` from an explicit owner stack through weak cross-stack references. This construct adds automatic regional placement and DNS checks. Its `V2` name follows CDK's earlier `DnsValidatedCertificate`; this library has no V1.

## CloudFront certificate in `us-east-1`

CloudFront requires a viewer certificate in [`us-east-1`](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html). This example puts the application in `eu-central-1` and lets the construct create a certificate owner in `us-east-1`.

```typescript
import { App, Stack } from 'aws-cdk-lib';
import { Distribution } from 'aws-cdk-lib/aws-cloudfront';
import { HttpOrigin } from 'aws-cdk-lib/aws-cloudfront-origins';
import { HostedZone } from 'aws-cdk-lib/aws-route53';
import { DnsValidatedCertificateV2 } from '@open-constructs/aws-cdk/aws-certificatemanager';

const app = new App();
const application = new Stack(app, 'Application', {
  env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: 'eu-central-1' },
});
const zone = HostedZone.fromHostedZoneAttributes(application, 'Zone', {
  hostedZoneId: 'Z1234567890',
  zoneName: 'example.com',
});
const certificate = new DnsValidatedCertificateV2(application, 'ViewerCertificate', {
  domainName: 'www.example.com',
  hostedZone: zone,
  certificateRegion: 'us-east-1',
});

new Distribution(application, 'Distribution', {
  certificate,
  domainNames: ['www.example.com'],
  defaultBehavior: { origin: new HttpOrigin('origin.example.com') },
});
```

Replace the example zone ID, domain, and origin. The zone must be public, delegated, and in the certificate account. Importing it does not verify live DNS or ownership.

## Placement and DNS

Omit both placement properties to keep the certificate in the containing stack, including an environment-agnostic stack. Set `certificateRegion` for an automatic owner in another concrete region. Use `certificateStack` for an explicit owner with its own name, synthesizer, tags, or lifecycle; that stack's region determines the certificate region. The two properties cannot be combined. A separate owner must be in the same app or stage, account, and partition.

Supply exactly one of `hostedZone` and `hostedZonesByDomain`. The first validates the primary name and every SAN in one zone. The second requires an exact zone entry for each primary and SAN name, including distinct apex and wildcard keys. Names are matched case-insensitively without a trailing dot; duplicate and out-of-zone names are rejected. An imported zone's public status and real account ownership remain caller preconditions.

Single-zone validation accepts fixed-length SAN arrays whose values resolve during synthesis. A list whose length is unknown until deployment cannot provide ACM's per-name validation options. Multi-zone validation requires concrete names.

## References and lifecycle

The construct sets weak cross-stack reference strength on its native certificate resource, even when the certificate stays in the containing stack. This overrides the app's `@aws-cdk/core:defaultCrossStackReferences` policy for other stacks that reference this certificate, including stacks in the same region. Remote consumers use `Fn::GetStackOutput`; owner-local consumers use a native reference. A nested owner serves only consumers in its own top-level stack tree; use a top-level `certificateStack` for external sharing.

Weak references allow the owner to change independently, so update consumers before deleting or replacing an in-use certificate. Changing the owner alone does not refresh a deployed consumer. ACM validation CNAMEs can be shared and are not removed by this construct. See [weak reference semantics](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getstackoutput.html) and [ACM DNS validation](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html).

The construct implements `ICertificate`, supports `Tags.of(certificate)`, `removalPolicy`, and `metricDaysToExpiry()`, and exposes `certificateResource` for native overrides. `fromCertificateAttributes()` imports an existing ARN without creating resources. The library requires Node.js 22+, `aws-cdk-lib` 2.268.0+, and `constructs` 10.8.1+.

For contributor validation, see the [integration fixture](../../test/aws-certificatemanager/README.md).
