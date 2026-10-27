# storr-cert-infra

CloudFormation stack that provisions the **single shared** ACM certificate for
`storr.build` with wildcard SANs and automated DNS validation. This is the sole
owner of the `storr.build` certificate; the `storr-infra` app and the
`storr-saas-infra` marketing site both consume it instead of minting their own.
See `docs/adr/0001-shared-storr-build-certificate.md`.

## Why this stack is standalone

The certificate covers names used by two independent apps: `storr.build` and
`www.storr.build` (the SaaS marketing site) and `app.`, `api.`, `*.preview.`
(the app). A wildcard and its base domain share one deterministic DNS validation
record, so more than one CloudFormation certificate for `storr.build` validating
into the same hosted zone collide (Route 53 `invalid set of changes`). Keeping
one certificate in a shared stack avoids the collision, and keeping it out of
either app's stack keeps the SaaS and app infrastructure decoupled.

All consumers deploy to `us-east-1`, so this one certificate serves CloudFront
(which requires `us-east-1`) as well as the app's regional ALB and API Gateway.

## Resources

| Resource | Type | Description |
|----------|------|-------------|
| `StorrBuildWildCardCertificate` | `AWS::CertificateManager::Certificate` | Certificate for `storr.build`, `*.storr.build`, and `*.preview.storr.build` |
| `CertArnParam` | `AWS::SSM::Parameter` | Publishes the certificate ARN to `/storr/acm/cert-arn` |

## Outputs

| Name | Description |
|------|-------------|
| `/storr/acm/cert-arn` (SSM) | **Primary contract.** Resolved by consumers via `{{resolve:ssm:/storr/acm/cert-arn}}` |
| `StorrBuildWildCard-CertificateArn` (CFN export) | Retained for backward compatibility |

## Consumers

| Stack | Resource | How it reads the cert |
|-------|----------|-----------------------|
| `storr-infra` | ALB listener (`40`), API Gateway domain (`45`), CloudFront (`50`) | `{{resolve:ssm:/storr/acm/cert-arn}}` |
| `storr-saas-infra` | CloudFront (`10-infra.yaml`) | `{{resolve:ssm:/storr/acm/cert-arn}}` |

## Parameters

| Parameter | Description |
|-----------|-------------|
| `StorrBuildHostedZoneId` | The Route 53 Hosted Zone ID for the `storr.build` domain |

## Files

- **infra.yaml** - CloudFormation template
- **deployment-file.yaml** - Deployment configuration (stack parameters)

## Deployment

Deployed via CloudFormation Git Sync from the AWS Console, using the shared
`CloudFormationGitSyncRole` (which already grants SSM parameter management under
`/storr/*`). **Deploy this stack before `storr-infra` and `storr-saas-infra`**,
which fail to resolve `/storr/acm/cert-arn` if it does not yet exist.

### First-time cutover from per-stack certificates

If a `storr.build` certificate is currently owned by another stack (the app's
old `05-foundation` cert, or the SaaS site's own cert), its DNS validation
record occupies the hosted zone and **this stack cannot create the shared
certificate until that record is free**. A `storr.build` validation record can
be managed by only one CloudFormation certificate at a time.

1. Diagnose current state:

   ```bash
   aws acm list-certificates --region us-east-1 \
     --query "CertificateSummaryList[?DomainName=='storr.build'].[CertificateArn,Status]" --output table
   aws route53 list-resource-record-sets --hosted-zone-id <ZONE_ID> \
     --query "ResourceRecordSets[?starts_with(Name,'_') && Type=='CNAME']" --output json
   ```

2. Free the validation record by removing the other stacks' certificates:
   - **App:** redeploy `storr-infra/05-foundation.yaml` with its `WildcardCertificate` removed (already done in that repo). Its certificate was not yet in use, so it deletes cleanly.
   - **SaaS (if its CloudFront is live on its own cert):** an ACM certificate in use by CloudFront cannot be deleted. Repoint SaaS CloudFront to the shared cert only **after** this stack publishes `/storr/acm/cert-arn`. Because every `storr.build` validation record is identical (same name and value), the cleanest path is: delete the SaaS `Certificate` during a brief window, deploy this stack, then redeploy SaaS pointing at `/storr/acm/cert-arn`. Confirm the live/traffic situation before choosing the window.

3. Deploy this stack, then `storr-infra`, then `storr-saas-infra`.

### Known gotcha

ACM does **not** remove a certificate's DNS validation CNAME from Route 53 when
the certificate is deleted. If this stack is ever torn down, delete the
`_<hash>.storr.build` validation record manually, or a future certificate for
the same domain will fail to validate. See
`../storr-infra/docs/troubleshooting/2026-10-12T09-59-13-acm-dns-validation-cname-conflict.md`.
