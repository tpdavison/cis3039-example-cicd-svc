# Product service — CI/CD example

A TypeScript Product API built with Azure Functions, Bicep infrastructure and a GitHub Actions build-and-deploy workflow. The service demonstrates application layers, input validation, dependency injection and automated deployment to a separate Test environment using a small in-memory data store.

## Development environment

The repository includes a [Dev Container configuration](.devcontainer/devcontainer.json) using the course image `ghcr.io/tpdavison/devops-labs:2026-latest`.

To use it, install Docker, Visual Studio Code and the Dev Containers extension. Start Docker, open your local clone in VS Code, and select **Dev Containers: Reopen in Container** from the Command Palette. Run the commands below in the container's terminal, from the repository root.

For course work, use your own private copy of the repository. Its name can be chosen freely.

If working outside the container, install Node.js 22, npm and Azure Functions Core Tools v4. The HTTP examples below use `curl` and shell line continuations supported by the container terminal.

## Run locally

Install the exact dependency versions recorded in the lockfile:

```bash
npm ci
```

For a new clone, create the local settings file:

```bash
cp local.settings.template.json local.settings.json
```

If `local.settings.json` already exists, keep your existing settings. It is ignored by Git and must not be committed. The template sets `FUNCTIONS_WORKER_RUNTIME` to `node` and leaves `AzureWebJobsStorage` empty.

Start the service:

```bash
npm start
```

The `prestart` script cleans `dist` and compiles the TypeScript before starting the Functions host. The default local address is `http://localhost:7071`. Keep this terminal running and use another terminal for HTTP requests. Stop the host with **Ctrl+C**; run `npm start` again to rebuild and restart after source changes.

### Local host storage

The Product repository stores data in memory and needs no Azure account. The HTTP endpoints can run locally with the template's empty `AzureWebJobsStorage` value, although the Functions host can report storage health warnings.

To configure local host storage, run the Azurite emulator in a separate container terminal:

```bash
npx --yes azurite --location /tmp/cis3039-azurite
```

This downloads and runs Azurite without adding it to the project's dependencies. Set `AzureWebJobsStorage` in your local settings file to `UseDevelopmentStorage=true`, then restart the Functions host. Keep Azurite running while using that setting, and stop it with **Ctrl+C** afterwards. The `/tmp` location is disposable emulator data; it does not persist the Product repository.

See Microsoft's [Azurite documentation](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite) and [connection string guidance](https://learn.microsoft.com/en-us/azure/storage/common/storage-connect-azurite) for details.

## HTTP API

All endpoints use anonymous access: a Function key or sign-in is not required. This is a demonstration service with synthetic data and an in-memory repository.

| Method | Path | Successful response |
| --- | --- | --- |
| `GET` | `/api/health/live` | HTTP 200 with `{"status":"live"}` |
| `GET` | `/api/products` | HTTP 200 with an array of products |
| `POST` | `/api/products` | HTTP 200 with the created or updated product |
| `PUT` | `/api/products` | HTTP 200 with the created or updated product |

Both write methods upsert a complete product: an existing ID is replaced, and a new ID is added. They use the ID in the JSON body, rather than an ID in the URL.

### Check liveness

```bash
curl -i http://localhost:7071/api/health/live
```

Expect HTTP 200 and `{"status":"live"}`. This checks that the Function can respond; it does not check storage readiness, dependencies or business correctness.

### List products

```bash
curl -i http://localhost:7071/api/products
```

A fresh process starts with two products, `p-001` and `p-002`. Each product contains `id`, `name`, `pricePence`, `description` and an ISO 8601 `updatedAt` timestamp.

### Create or update a product

Use the supplied sample body:

```bash
curl -i -X POST http://localhost:7071/api/products \
  -H "Content-Type: application/json" \
  --data-binary @samples/product-post.json
```

Expect HTTP 200 with the saved product. Repeat the list request to see `product-123`. Replace `POST` with `PUT` to use the other supported write method.

The JSON body must contain:

| Field | Requirement |
| --- | --- |
| `id` | A string containing at least one non-whitespace character |
| `name` | A string containing at least one non-whitespace character |
| `pricePence` | A non-negative safe integer, representing the price in pence |
| `description` | A string containing at least one non-whitespace character |

The application supplies `updatedAt` when saving; clients do not need to send it. Malformed JSON, missing fields, incorrect types and invalid values return HTTP 400. Unexpected server failures return HTTP 500 with a generic message rather than internal exception details.

Try the invalid-price sample:

```bash
curl -i -X POST http://localhost:7071/api/products \
  -H "Content-Type: application/json" \
  --data-binary @samples/product-post-invalid.json
```

Expect HTTP 400 because the price is negative.

### Data lifetime

Products are held in memory in each worker process. Restarting the process restores the seed data and discards changes. Different worker processes have independent stores. Configuring host storage does not change this behaviour.

## Continuous integration and deployment

The [Build and Deploy to Test workflow](.github/workflows/build-and-deploy.yml) runs on:

- A push to `main`.
- A pull request targeting `main`.
- A manual run using `workflow_dispatch`.

The workflow has two jobs, both running on `ubuntu-latest`:

1. **`build-and-package`** checks out the code, selects Node.js 22 with npm caching, runs `npm ci` and `npm run build`, assembles the Function App package and uploads it as `functionapp-package`. The artefact is retained for one day.
2. **`deploy-to-test`** depends on the first job, downloads its package and deploys it to the configured Azure Function App using a publishing profile.

| Event/ref | Build and package | Deploy to Test |
| --- | --- | --- |
| Push to `main` | Yes | After success |
| Pull request targeting `main` | Yes | No |
| Manual run on `main` | Yes | After success |
| Manual run on another branch | Yes | No |

A failed build, package assembly or upload blocks deployment. Ordinary pushes to other branches do not trigger this workflow. Manual dispatch requires the workflow to exist on the repository's default branch; see [GitHub's manual-run guidance](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow).

The build compiles TypeScript into `dist` with strict type checking. This is build validation, not an automated unit test or deployed smoke test. Repository permissions are limited to `contents: read`; Azure authentication uses the separate publishing-profile secret.

### Publishable package

The assembly step creates a fresh `package` directory containing:

```text
package/
  host.json
  package.json
  package-lock.json
  dist/
    src/
      functions/
        health-live-http.js
        list-products-http.js
        upsert-product-http.js
      ...
  node_modules/
```

It copies the metadata and complete compiled tree, then runs `npm ci --omit=dev --prefix package` to install locked production dependencies. Local settings and credentials are not copied. The upload contains the contents of `package`, with `host.json` at the archive root. The deployment job uses that package without rebuilding it.

The workflow deploys application code to existing resources. Provision the Azure infrastructure using Bicep before running a deployment.

A local build check uses:

```bash
npm ci
npm run build
```

## Provision the Azure infrastructure

The [Bicep template](iac/main.bicep) provisions a `Standard_LRS` Storage Account and a Windows Function App configured for Functions v4, Node.js 22, a 64-bit worker and run-from-package deployment. It supplies the host/content storage connections, lets Azure generate the content share name, enables SCM publishing-profile authentication and disables FTP publishing. Its outputs contain the app name, public base URL and hosting-plan resource ID, with no credentials.

Azure creates or selects a classic Consumption hosting plan when the Function App is provisioned. The hosting checks below confirm that the plan uses SKU `Y1` and tier `Dynamic`. See [Azure Functions infrastructure guidance](https://learn.microsoft.com/en-us/azure/azure-functions/functions-infrastructure-as-code?pivots=consumption-plan).

### Select the subscription and names

Run these Bash commands from the repository root inside the Dev Container. The container includes Azure CLI. Sign in, select your subscription and confirm the account:

```bash
az login --use-device-code
```

```bash
az account set --subscription 'YOUR_SUBSCRIPTION_ID'
```

```bash
az account show --query '{name:name,id:id,tenantId:tenantId}' --output table
```

Replace the region placeholder with a region permitted by your subscription and supporting Windows Consumption Functions. Replace `ab47` with your own short lowercase-letter-and-digit identifier. Function App and Storage Account names must be globally unique; storage names allow only 3–24 lowercase letters and digits. These Azure naming examples do not constrain your GitHub repository name.

```bash
LAB_REGION='YOUR_PERMITTED_REGION'
LAB_GROUP='products-test-ab47-rg'
FUNCTION_APP_NAME='products-test-ab47-func'
STORAGE_ACCOUNT_NAME='productstestab47store'
```

Use a fresh, dedicated disposable resource group. Check whether the chosen name already exists:

```bash
az group exists --name "$LAB_GROUP"
```

Expect `false`. If it is `true`, inspect the group before proceeding and choose a different name if it contains assessment, shared or unrelated resources.

```bash
az group create --name "$LAB_GROUP" --location "$LAB_REGION" --output none
```

Keep these variables in the same terminal. In a new terminal, reselect the subscription and restore the variables before running further Azure commands.

### Compile, validate and provision

Install the Bicep compiler if it is not already available:

```bash
az bicep install
```

```bash
az bicep build --file iac/main.bicep --outfile /tmp/cis3039-main.json
```

Validate the deployment against the selected group:

```bash
az deployment group validate \
  --resource-group "$LAB_GROUP" \
  --template-file iac/main.bicep \
  --parameters functionAppName="$FUNCTION_APP_NAME" storageAccountName="$STORAGE_ACCOUNT_NAME" location="$LAB_REGION" \
  --output none
```

If validation succeeds, provision the resources. This creates Azure resources that can incur charges:

```bash
az deployment group create \
  --name products-test \
  --resource-group "$LAB_GROUP" \
  --template-file iac/main.bicep \
  --parameters functionAppName="$FUNCTION_APP_NAME" storageAccountName="$STORAGE_ACCOUNT_NAME" location="$LAB_REGION" \
  --query properties.outputs --output json
```

Record the non-secret outputs. `deployedFunctionAppName` is the value needed by the workflow. `functionAppUrl` is the API base URL; the app will not serve these Functions until application code has been deployed.

### Check the hosting configuration

```bash
PLAN_ID=$(az functionapp show --resource-group "$LAB_GROUP" --name "$FUNCTION_APP_NAME" --query serverFarmId --output tsv)
```

```bash
az appservice plan show --ids "$PLAN_ID" \
  --query '{name:name,sku:sku.name,tier:sku.tier,linux:reserved}' --output table
```

Expect SKU `Y1`, tier `Dynamic` and `linux` set to `false`. Stop and investigate any other plan before deploying application code.

```bash
az functionapp config show --resource-group "$LAB_GROUP" --name "$FUNCTION_APP_NAME" \
  --query '{use32BitWorkerProcess:use32BitWorkerProcess}' --output table
```

Expect `false`. Inspect only the non-secret runtime settings:

```bash
az functionapp config appsettings list --resource-group "$LAB_GROUP" --name "$FUNCTION_APP_NAME" \
  --query "[?name=='FUNCTIONS_EXTENSION_VERSION' || name=='FUNCTIONS_WORKER_RUNTIME' || name=='WEBSITE_NODE_DEFAULT_VERSION' || name=='WEBSITE_RUN_FROM_PACKAGE'].{setting:name,value:value}" \
  --output table
```

Expect `~4`, `node`, `~22` and `1` respectively. To check that both storage settings have values without displaying their connection strings:

```bash
az functionapp config appsettings list --resource-group "$LAB_GROUP" --name "$FUNCTION_APP_NAME" \
  --query "[?name=='AzureWebJobsStorage' || name=='WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'].{setting:name,configured:value!=''}" \
  --output table
```

Expect two entries with `configured` set to `true`.

## Configure GitHub deployment

Ensure Actions is enabled in your private repository. Under **Settings → Secrets and variables → Actions**, configure:

| Kind | Name | Value |
| --- | --- | --- |
| Repository variable | `AZURE_FUNCTIONAPP_NAME` | The exact deployed Function App name |
| Repository secret | `AZURE_FUNCTIONAPP_PUBLISH_PROFILE` | The complete publishing-profile XML for that Function App |

In the Azure portal, open the Function App's **Overview**, select **Get publish profile**, and download the profile. Open it locally and paste the complete XML into the GitHub secret. The template enables SCM basic authentication because this deployment method requires it; a subscription policy can still block it. FTP publishing remains disabled. See the action's [publishing-profile instructions](https://github.com/Azure/functions-action/blob/master/docs/authentication.md).

The profile is a deployment credential, not an API access key or Storage connection string. Keep it out of Git, screenshots and logs, and remove the downloaded file once the secret is stored. Runtime storage settings are configured by Bicep; they do not belong in this GitHub secret or in `local.settings.template.json`.

## Deploy and verify

Commit and push the application and workflow to `main` in your own repository, or manually run **Build and Deploy to Test** on `main`. Open its run in **Actions** and inspect both jobs. A PR run should complete build/package validation with the deployment job skipped.

After successful deployment, obtain the hostname:

```bash
FUNCTION_APP_HOST=$(az functionapp show --resource-group "$LAB_GROUP" --name "$FUNCTION_APP_NAME" --query defaultHostName --output tsv)
```

```bash
curl -i "https://$FUNCTION_APP_HOST/api/health/live"
```

Expect HTTP 200 with `{"status":"live"}`. Then verify a business endpoint:

```bash
curl -i "https://$FUNCTION_APP_HOST/api/products"
```

Expect HTTP 200 with product data. Use the valid and invalid POST samples from the local API examples against this hostname to verify successful writes and HTTP 400 validation responses. Liveness alone does not prove business correctness or storage readiness.

To confirm a new version reached Azure, make a harmless distinguishable change, such as changing a seeded product description, commit and push to `main`, then inspect the new workflow run and deployed response. Capture the successful jobs, target URL and HTTP results before cleanup.

If deployment succeeds but requests fail, inspect the Function App's registered Functions and runtime configuration in the portal. Confirm the package has `host.json` at its root and JavaScript under `dist/src/functions`, matching `package.json`'s `main` pattern. Deployment success alone does not prove that Functions were indexed or that the service works.

The deployed endpoints are public and anonymous. Publishing credentials control deployment access; they do not restrict callers. Browser clients also need their exact origin allowed in the Function App's CORS configuration.

## Clean up disposable resources

After verification and evidence capture, retire the dedicated lab environment. First remove the deployment job from the workflow, rename the workflow to describe build/package validation, and commit that change so later pushes cannot redeploy to deleted resources. Remove `AZURE_FUNCTIONAPP_PUBLISH_PROFILE` and `AZURE_FUNCTIONAPP_NAME` from the repository settings. Keep the build/package job available for CI.

Confirm the active subscription, target group and all resources before deletion:

```bash
az account show --query '{name:name,id:id}' --output table
```

```bash
az group show --name "$LAB_GROUP" --query '{name:name,id:id,location:location}' --output table
```

```bash
az resource list --resource-group "$LAB_GROUP" --query '[].{name:name,type:type}' --output table
```

Delete only the dedicated disposable group after confirming it has no assessment, shared or unrelated dependencies. An implicitly selected plan may be shared or belong to another group; do not delete it separately without checking its ownership and other apps.

```bash
az group delete --name "$LAB_GROUP"
```

The command asks for confirmation and waits for deletion. Verify removal:

```bash
az group exists --name "$LAB_GROUP"
```

Expect `false`. Deleting the Function App retires its publishing credentials; if you retain the app instead, reset its publishing profile to invalidate the old credential. Local development and build-only CI do not require the deleted Azure resources.

## npm commands

| Command | Purpose |
| --- | --- |
| `npm run typecheck` | Check TypeScript without emitting JavaScript |
| `npm run build` | Compile TypeScript into `dist` |
| `npm run watch` | Recompile when TypeScript files change |
| `npm run clean` | Remove `dist` |
| `npm start` | Clean, build and start the local Functions host |
| `npm test` | Print that no automated tests are configured and exit with status 1 |

There is no automated test suite in this example. The HTTP requests above provide manual checks.

## Source structure

| Directory | Responsibility |
| --- | --- |
| `src/domain` | Product model, validation and repository interface |
| `src/app` | List and upsert use cases, with typed results and injected dependencies |
| `src/infra` | Fake in-memory Product repository |
| `src/config` | Composition root supplying the repository and application clock |
| `src/functions` | HTTP triggers, request parsing, response mapping and safe error boundaries |
| `samples` | Valid and invalid product request bodies |
| `iac` | Bicep infrastructure for the Test Function App and host storage |
| `.github/workflows` | Build/package validation and conditional Test deployment |

Generated output in `dist`, installed dependencies in `node_modules`, and `local.settings.json` are ignored by Git. Application configuration templates and the dependency lockfile are committed so a new clone can reproduce the setup.
