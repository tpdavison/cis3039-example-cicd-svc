# Product service — CI example

A TypeScript Product API built with Azure Functions and a build-only GitHub Actions workflow with optional publishable package upload. The service demonstrates application layers, input validation, dependency injection and continuous integration using a small in-memory data store.

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

| Method | Path               | Successful response                          |
| ------ | ------------------ | -------------------------------------------- |
| `GET`  | `/api/health/live` | HTTP 200 with `{"status":"live"}`            |
| `GET`  | `/api/products`    | HTTP 200 with an array of products           |
| `POST` | `/api/products`    | HTTP 200 with the created or updated product |
| `PUT`  | `/api/products`    | HTTP 200 with the created or updated product |

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

| Field         | Requirement                                                  |
| ------------- | ------------------------------------------------------------ |
| `id`          | A string containing at least one non-whitespace character    |
| `name`        | A string containing at least one non-whitespace character    |
| `pricePence`  | A non-negative safe integer, representing the price in pence |
| `description` | A string containing at least one non-whitespace character    |

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

## Continuous integration

The [Build workflow](.github/workflows/build.yml) runs on:

- A push to `main`.
- A pull request targeting `main`.
- A manual run using `workflow_dispatch`.

Its single job runs on `ubuntu-latest`, checks out the code, selects Node.js 22 with npm caching, installs dependencies using `npm ci`, and runs `npm run build`. Repository permissions are limited to `contents: read`.

The build compiles the TypeScript into `dist` with strict type checking. A compilation error fails the job. This is build validation; it does not demonstrate that the application's behaviour is correct. The workflow has no deployment steps or Azure credential requirements.

Ensure GitHub Actions is enabled in your repository. After pushing a commit to `main`, open the repository's **Actions** tab, select **Build**, and inspect the job and step logs. A local check using the same commands is:

```bash
npm ci
npm run build
```

### Optional Function App package

The manual trigger accepts a boolean input, `upload_artifact`, labelled **Upload Function App package**, which defaults to `false`. Set it to `true` to assemble and upload a publishable Azure Functions package after a successful build. Push and pull-request runs only build; manual runs with the input left off also only build. A failed build prevents packaging and upload, and a packaging failure prevents upload.

To request an upload, open **Actions → Build → Run workflow**, select a branch containing this workflow version, enable **Upload Function App package**, and start the run. GitHub requires the workflow to exist on the repository's default branch for [manual dispatch](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow) to be available.

The packaging step creates a fresh `package` directory with this layout:

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

It copies the host configuration, package metadata and complete compiled `dist` tree, then runs `npm ci --omit=dev --prefix package` to install the locked runtime dependencies. Development tools such as TypeScript and `rimraf` are excluded from the installed dependencies. Local settings and credentials are not copied.

After the run succeeds, open its summary and download `functionapp-package` from the **Artifacts** section. The archive contains the contents of `package`, with `host.json` at its root. It is retained for one day. A requested upload fails if no package files are found.

This artefact contains the application code and runtime dependencies for deployment to an appropriately configured Azure Function App. Producing the package does not provision infrastructure or deploy the service; this workflow only builds, packages and uploads it.

## npm commands

| Command             | Purpose                                                             |
| ------------------- | ------------------------------------------------------------------- |
| `npm run typecheck` | Check TypeScript without emitting JavaScript                        |
| `npm run build`     | Compile TypeScript into `dist`                                      |
| `npm run watch`     | Recompile when TypeScript files change                              |
| `npm run clean`     | Remove `dist`                                                       |
| `npm start`         | Clean, build and start the local Functions host                     |
| `npm test`          | Print that no automated tests are configured and exit with status 1 |

There is no automated test suite in this example. The HTTP requests above provide manual checks.

## Source structure

| Directory       | Responsibility                                                             |
| --------------- | -------------------------------------------------------------------------- |
| `src/domain`    | Product model, validation and repository interface                         |
| `src/app`       | List and upsert use cases, with typed results and injected dependencies    |
| `src/infra`     | Fake in-memory Product repository                                          |
| `src/config`    | Composition root supplying the repository and application clock            |
| `src/functions` | HTTP triggers, request parsing, response mapping and safe error boundaries |
| `samples`       | Valid and invalid product request bodies                                   |

Generated output in `dist`, installed dependencies in `node_modules`, and `local.settings.json` are ignored by Git. Application configuration templates and the dependency lockfile are committed so a new clone can reproduce the setup.
