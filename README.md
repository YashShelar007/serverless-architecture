# serverless-architecture

A three-endpoint REST API (`/hello`, `/users`, `/stats`) on Google Cloud Functions (1st gen), fronted by API Gateway, with one function also described in a Deployment Manager config. It shows the smallest working path from function code to a single public base URL on GCP.

## What it does not do

- No database or state. Every response is hard-coded.
- No authentication. The functions are deployed with `--allow-unauthenticated` and the gateway spec defines no security scheme.
- No tests and no CI.
- Deployment Manager covers only `hello_world`; the other two functions are deployed with `gcloud`.

## Quickstart

Run a function locally (verified, Python 3, no dependencies):

```bash
cd functions/users
python3 -c "import main; print(main.get_users(None))"
# ({'users': ['alice', 'bob', 'carol']}, 200)
```

`functions/hello` (`hello_world`) and `functions/stats` (`get_stats`) work the same way.

Deploying to GCP (not verified in this cleanup: needs a billed GCP project, which was not available). Enable the Cloud Functions, API Gateway and Deployment Manager APIs, then from the repo root:

```bash
gcloud functions deploy hello_world --runtime python311 --trigger-http --entry-point hello_world \
  --allow-unauthenticated --region us-central1 --source=functions/hello --no-gen2
# repeat for get_users (functions/users) and get_stats (functions/stats)

# edit the three address: lines in apigateway/api-config.yaml to your project, then
gcloud api-gateway apis create serverless-api --project=YOUR_PROJECT_ID
gcloud api-gateway api-configs create serverless-api-config --api=serverless-api \
  --openapi-spec=apigateway/api-config.yaml --project=YOUR_PROJECT_ID
gcloud api-gateway gateways create serverless-gateway --api=serverless-api \
  --api-config=serverless-api-config --location=us-central1 --project=YOUR_PROJECT_ID
```

The gateway prints a default hostname; `GET /hello`, `/users` and `/stats` on it reach the three functions.

## How it works

```
client -> API Gateway (Swagger 2.0 spec) -+- /hello -> Cloud Function hello_world
                                          +- /users -> Cloud Function get_users
                                          +- /stats -> Cloud Function get_stats
```

- `functions/<name>/main.py` holds one HTTP function each. They return a string (`hello_world`) or a dict that the Functions framework serializes as JSON (`get_users`, `get_stats`). Each `requirements.txt` is empty apart from a comment.
- `apigateway/api-config.yaml` is a Swagger 2.0 spec. Each path uses `x-google-backend` to point at the matching function's `cloudfunctions.net` URL.
- `deployment-manager/deployment.yaml` declares `hello_world` as a Cloud Functions resource (256 MB, 60 s timeout, source from a Cloud Storage zip). `functions/hello/hello_source.zip` is a copy of that source.

## Status

Built in 2025 as a cloud computing project. Archived: no further changes planned.

## Known limits

- The spec and the Deployment Manager config contain the original author's project ID (`severless-architecture`, spelled as in the config) and bucket name (`serverless-api-hello-bucket`). Replace both before deploying.
- Python 3.11 and 1st-gen Cloud Functions were current when this was written. Not checked against today's runtime support.

## License

MIT, see [LICENSE](LICENSE).
