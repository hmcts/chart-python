# chart-python

[![Build Status](https://dev.azure.com/hmcts/PlatformOperations/_apis/build/status/1229)](https://dev.azure.com/hmcts/PlatformOperations/_build?definitionId=1229&_a=summary)

This chart is intended for simple Python microservices.

We will take small PRs and small features to this chart but more complicated needs should be handled in your own chart.

*NOTE*: `/health/readiness` and `/health/liveness` are required endpoints. Your service must implement both. See the [Health Check Convention](#health-check-convention) section below.

This chart adds below templates from [chart-library](https://github.com/hmcts/chart-library/) based on the chosen configuration:

- [Deployment](https://github.com/hmcts/chart-library/tree/master#deployment)
- [Keyvault Secrets](https://github.com/hmcts/chart-library#keyvault-secret-csi-volumes)
- [Horizontal Pod Auto Scaler](https://github.com/hmcts/chart-library/tree/master#hpa-horizontal-pod-auto-scaler)
- [Ingress](https://github.com/hmcts/chart-library/tree/master#ingress)
- [Pod Disruption Budget](https://github.com/hmcts/chart-library/tree/master#pod-disruption-budget)
- [Service](https://github.com/hmcts/chart-library/tree/master#service)


## Health Check Convention

Your Python service must expose two health endpoints:

- `GET /health/readiness` — return HTTP 200 when the service is ready to receive traffic
- `GET /health/liveness` — return HTTP 200 when the service process is alive

Minimal FastAPI example:

```python
@app.get("/health/readiness")
async def readiness():
    return {"status": "UP"}

@app.get("/health/liveness")
async def liveness():
    return {"status": "UP"}
```

The readiness endpoint is the right place to add dependency checks (e.g. database connectivity). The liveness endpoint should remain simple — if it fails, the pod will be restarted.

## Example configuration

```yaml
applicationPort: 8000
environment:
  REFORM_TEAM: cnp
  REFORM_SERVICE_NAME: my-python-service
  REFORM_ENVIRONMENT: preview
configmap:
  VAR_A: VALUE_A
  VAR_B: VALUE_B
secrets:
  ENVIRONMENT_VAR:
      secretRef: some-secret-reference
      key: connectionString
  ENVIRONMENT_VAR_OTHER:
      secretRef: some-secret-reference-other
      key: connectionStringOther
      disabled: true #ENVIRONMENT_VAR_OTHER will not be set to environment
keyVaults:
  "my-vault":
    secrets:
      - my-secret-key
applicationInsightsInstrumentKey: "some-key"
```

If you wish to use pod identity for accessing the key vaults instead of a service principal you need to set a flag `aadIdentityName: <identity-name>`
e.g.
```yaml
aadIdentityName: my-service
keyVaults:
  "my-vault":
    usePodIdentity: true
    secrets:
      - my-secret-key
```

## Startup probes

Startup probes are defined in the [library template](https://github.com/hmcts/chart-library/tree/master#startup-probes) and should be configured for slow starting applications.
The default values below (defined in the chart) should be sufficient for most Python applications:

```yaml
startupPath: '/health/liveness'
startupDelay: 10
startupTimeout: 3
startupPeriod: 10
startupFailureThreshold: 10
```

To configure startup probes for a slow starting application:
- Set the value of `(startupFailureThreshold x startupPeriod)` to cover the longest startup time required by the application
- If `livenessDelay` is currently configured, set the value to `0`

### HPA Horizontal Pod Autoscaler

To adjust the number of pods in a deployment depending on CPU/memory utilization, AKS supports horizontal pod autoscaling. HPA is enabled by default.

```yaml
autoscaling:        # Default is true
  enabled: true
  maxReplicas: 5    # Optional, will use replicas + 2 if not set
  minReplicas: 2    # Optional, will use replicas if not set
  cpu:
    enabled: true
    averageUtilization: 80
  memory:
    enabled: true
    averageUtilization: 80
```

## Postgresql

If you need to use a Postgresql database for testing then you can enable it
by setting the following flag in your application config with:

```yaml
python:
  environment:
    DB_HOST: "{{ .Release.Name }}-postgresql"
    DB_USER_NAME: "{{ .Values.postgresql.auth.username}}"
    DB_PASSWORD: "{{ .Values.postgresql.auth.password}}"

postgresql:
  enabled: true
```

## Development and Testing

Default configuration (e.g. default image and ingress host) is set up for preview. This is suitable for local development and testing.

- Ensure you have logged in with `az cli` and are using `preview` subscription (use `az account show` to display the current one).
- For local development see the `Makefile` for available targets.
- To execute an end-to-end build, deploy and test run `make`.
- To clean up deployed releases, charts, test pods and local charts, run `make clean`

## Azure DevOps Builds

Builds are run against the CFT Preview AKS cluster (`cft-preview-00-aks` / `cft-preview-01-aks`), whichever is active.

### Pull Request Validation

A build is triggered when pull requests are created. This build will run `helm lint` and deploy the chart using `ci-values.yaml`.

### Release Build

Triggered when the repository is tagged (e.g. when a release is created). Also performs linting and testing, and will publish the chart to ACR on success.

### Releases

We use semantic versioning via GitHub releases to handle new releases of this application chart, this is done via automation called Release Drafter. When you merge a PR to main, a new draft release will be created.
More information is available about the [release process and how to create draft releases for testing purposes in more depth](https://hmcts.github.io/ops-runbooks/Testing-Changes/drafting-a-release.html)
