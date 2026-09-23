# Certificate integration fixture

This fixture requests a public certificate in `us-east-1`, attaches it to CloudFront from `eu-central-1`, and checks ACM issuance and the deployed viewer certificate. It needs an approved public Route 53 zone in the deployment account. The alias `ocf-certificate-integ.<zone name>` must be available.

Set `OCF_CERTIFICATE_INTEG_ZONE_ID` and `OCF_CERTIFICATE_INTEG_ZONE_NAME` to a disposable zone whose ID and name may appear in a public snapshot. Region-only stack environments keep the account ID out of the snapshot. An offline synthesis does not establish that DNS delegation or deployment works.

```bash
OCF_CERTIFICATE_INTEG_ZONE_ID=Z1234567890 OCF_CERTIFICATE_INTEG_ZONE_NAME=example.com CDK_OUTDIR=/tmp/ocf-certificate-integ-synth ./node_modules/.bin/ts-node --project tsconfig.dev.json test/aws-certificatemanager/integ.dns-validated-certificate-v2.ts
npm run integ -- --directory test/aws-certificatemanager --language typescript --list
```

After deployment and cleanup are authorized for the account and zone, verify the AWS identity, zone delegation, bootstrap status in both regions, existing CloudFront aliases, and the zone's current records. Run the fixture with the runner's cleanup enabled:

```bash
npm run integ:update -- --directory test/aws-certificatemanager --language typescript --parallel-regions us-east-1 --max-workers 1 --strict --disable-update-workflow integ.dns-validated-certificate-v2.ts
npm run integ -- --directory test/aws-certificatemanager --language typescript --strict integ.dns-validated-certificate-v2.ts
```

The `--disable-update-workflow` option applies only to the first run without a snapshot. Inspect the runner-generated snapshot for account IDs, zone identifiers, and local paths before committing it. Do not edit a snapshot into a claim of deployed evidence.

After the runner finishes, independently confirm that the three stacks, certificate, distribution, and assertion resources are gone and that the zone's records match the pre-run list. ACM validation CNAMEs can be shared; retain any record whose ownership is uncertain.
