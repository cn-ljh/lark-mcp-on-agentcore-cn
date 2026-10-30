# AWS China (aws-cn) deployment

This branch (`china-region`) can deploy the service to the AWS China regions: Ningxia `cn-northwest-1` and Beijing `cn-north-1`. Amazon Bedrock AgentCore Runtime is available in both regions, so the architecture is the same as the global version. Only the entry point and the partition-dependent parts are adapted.

Verified: full-stack deploy in Ningxia; Kiro connected through OAuth dynamic client registration and could call tools (2026-10-30).

## Differences from the global partition

| Item | Global (aws) | China (aws-cn) |
|-|-|-|
| Public entry | CloudFront → REST API | HTTP API (`$default` stage, no path prefix in the URL), no CloudFront |
| Protection | Optional WAFv2 (us-east-1, CLOUDFRONT scope) | No WAF. API Gateway throttling: 50 rps / burst 100 overall, 5 rps / burst 10 on `POST /register` |
| ARN partition | `arn:aws:` | `arn:aws-cn:` |
| Service DNS suffix | `amazonaws.com` | `amazonaws.com.cn` |
| AgentCore trust principal | `bedrock-agentcore.amazonaws.com` | Same. Writing it as `.com.cn` fails with `MalformedPolicyDocument` |
| Console links | `console.aws.amazon.com` | `console.amazonaws.cn` |

### Why China uses an HTTP API instead of a REST API

- CloudFront in China cannot serve on the default `*.cloudfront.cn` domain. It needs an ICP-filed domain and a certificate, which this project does not manage, so API Gateway is exposed directly.
- A REST API stage URL has a `/prod` prefix. MCP clients follow RFC 8414 / RFC 9728 and build `/register` and `/.well-known/*` from the origin root. API Gateway answers those requests itself with 403, so dynamic client registration fails (Kiro shows `Dynamic client registration rejected`).
- A REST API also renames the Lambda's `WWW-Authenticate` header to `x-amzn-Remapped-WWW-Authenticate`, so clients never see the 401 challenge.
- An HTTP API `$default` stage is served at the origin root and returns response headers unchanged, which removes both problems. The integrations use payload format v1.0, so the Lambdas receive the same event shape as with the REST proxy integration and need no code changes.
- China regions have no EDGE-optimized endpoints.

## Deploy

Prerequisites: an ARM64 build host, a China-partition credentials profile (`gcr` in the examples below), and a Feishu custom app created beforehand (see [app-setup_en.md](app-setup_en.md)).

```bash
git clone <this-repo> && cd <repo> && git checkout china-region
# first CDK use in the region (China does not need a us-east-1 bootstrap)
cd infra && AWS_PROFILE=gcr npx cdk bootstrap aws://<account>/cn-northwest-1 && cd ..
# interactive: pick "cn-northwest-1 Ningxia (aws-cn)" and brand "feishu"
AWS_PROFILE=gcr AWS_REGION=cn-northwest-1 bash scripts/deploy.sh
# redeploy with the saved config
AWS_PROFILE=gcr AWS_REGION=cn-northwest-1 bash scripts/deploy.sh --yes
```

> Set `AWS_REGION` as well when running non-interactively with `--yes`. Without a region, STS calls with China credentials go to the global endpoint and fail with `InvalidClientTokenId`, and the script then reports that credentials are not configured.

When the deploy finishes, the output gives:
- MCP endpoint: `https://<api-id>.execute-api.<region>.amazonaws.com.cn/mcp`
- Feishu redirect URL: `https://<api-id>.execute-api.<region>.amazonaws.com.cn/callback`. Add it in the Feishu app's Security Settings.

Client config: `{"type":"http","url":"https://<api-id>.execute-api.<region>.amazonaws.com.cn/mcp"}`

## Verify

```bash
B=https://<api-id>.execute-api.<region>.amazonaws.com.cn
curl -s -o /dev/null -w '%{http_code}\n' $B/.well-known/oauth-authorization-server   # 200
curl -s -o /dev/null -w '%{http_code}\n' $B/.well-known/oauth-protected-resource     # 200
curl -s -D - -o /dev/null -X POST $B/mcp -H 'content-type: application/json' -d '{}' | grep -i 'www-authenticate'  # 401 with resource_metadata
```

## Known limitations

- The entry point is the native `execute-api` domain, with no ICP filing. Evaluate for yourself whether it is reachable long term from mainland networks and whether it meets compliance requirements. A custom domain needs an ICP-filed domain configured as an API Gateway custom domain.
- There is no WAF, only API Gateway throttling.
- Building the image inside mainland China can be slow or fail for npm, GitHub, and awscli downloads. Build outside the mainland if you can.
- In China, AgentCore Memory and Gateway connectors are not available. This project only uses Runtime, so it is not affected.
