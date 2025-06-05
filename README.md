# Investigation Dossier Manager - Backend MVP

This is the NestJS backend for the Investigation Dossier Manager application, designed to run on Google Cloud Run.

## Prerequisites

* Node.js (LTS version, e.g., v18 or v20)
* pnpm (package manager): `npm install -g pnpm`
* Docker Desktop (for containerization)
* Google Cloud SDK (`gcloud` CLI): Authenticated and configured for your Google Cloud project.
* Git
* A code editor (e.g., VS Code)

---

## I. Local Development Setup

### 1. Clone the Repository

If you haven't already, clone the repository:
```bash
git clone <YOUR_REPOSITORY_URL>
cd <YOUR_REPOSITORY_DIRECTORY>
```

### 2. Install Dependencies

```powershell
pnpm install
```

### 3. Set Up Environment Variables

Create a `.env` file in the project root by copying the example file.

```powershell
# For PowerShell
Copy-Item .env.example .env

# For bash/WSL
# cp .env.example .env
```

Edit the `.env` file with your specific development values:

* **`DATABASE_URL`**: See "Connecting to Cloud SQL Locally" below.
* **`JWT_SECRET`**: Generate a strong, random string for signing tokens.
* **`GCS_SA_KEY_JSON_STRING`**: **(Security Warning)** For local development, it's safer to use Application Default Credentials by running `gcloud auth application-default login`. Pasting a JSON key string into a `.env` file is risky and should be avoided. For deployed environments, always use a secret manager.
* **Other variables**: `GCS_BUCKET_NAME`, `GCS_PROJECT_ID`, `FRONTEND_ORIGIN_URL`, and `PORT` should be configured to match your specific GCP and frontend setup.

### 4. Connecting to Cloud SQL Locally (Recommended)

To securely connect to your Cloud SQL PostgreSQL instance from your local machine, use the Cloud SQL Auth Proxy.

**a. Install Cloud SQL Auth Proxy:** Follow the official [Google Cloud Documentation](https://cloud.google.com/sql/docs/postgres/sql-proxy).

**b. Get your Cloud SQL Instance Connection Name:**
You can find this in the GCP console or by using the `gcloud` CLI:
```powershell
gcloud sql instances describe <YOUR_CLOUD_SQL_INSTANCE_NAME> --project=<YOUR_GCP_PROJECT_ID> --format="value(connectionName)"
```
It will look like: `<YOUR_GCP_PROJECT_ID>:<YOUR_REGION>:<YOUR_CLOUD_SQL_INSTANCE_NAME>`

**c. Start the Cloud SQL Auth Proxy:**
Open a new terminal and run the command with your instance connection name.
```powershell
# If you downloaded the executable directly
# ./cloud-sql-proxy <INSTANCE_CONNECTION_NAME>

# If you installed it on your system's PATH
cloud-sql-proxy <INSTANCE_CONNECTION_NAME>
```
The proxy will listen on `127.0.0.1:5432` by default for PostgreSQL.

**d. Configure `DATABASE_URL` in your `.env` file:**
Use the proxy's local address. The proxy provides the secure tunnel, so `sslmode=disable` is appropriate here.
```
DATABASE_URL=postgresql://<DB_USER>:<DB_PASSWORD>@127.0.0.1:5432/<DB_NAME>?sslmode=disable
```

### 5. Run the Application

```powershell
pnpm run start:dev
```
The application will be available at `http://localhost:PORT` (e.g., `http://localhost:3000`).

---

## II. Building the Application

To create a production-ready build in the `dist` folder:
```powershell
pnpm run build
```

---

## III. Docker Operations

### 1. Build the Docker Image
Ensure Docker Desktop is running. Tag the image for your Google Artifact Registry.

```powershell
$IMAGE_TAG = "latest" # Or a semantic version like "v1.0.0"
$ARTIFACT_REGISTRY_URL = "<REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT_ID>/<YOUR_REPO_NAME>"
$IMAGE_NAME_WITH_TAG = "${ARTIFACT_REGISTRY_URL}/<YOUR_IMAGE_NAME>:${IMAGE_TAG}"

docker build -t $IMAGE_NAME_WITH_TAG .
Write-Host "Built image: $IMAGE_NAME_WITH_TAG"
```

### 2. Push Docker Image to Google Artifact Registry

**a. Authenticate Docker with Artifact Registry (if needed):**
```powershell
gcloud auth configure-docker <REGION>-docker.pkg.dev --project=<YOUR_GCP_PROJECT_ID>
```
**b. Push the image:**
```powershell
docker push $IMAGE_NAME_WITH_TAG
Write-Host "Pushed image: $IMAGE_NAME_WITH_TAG"
```

---

## IV. Deploy to Google Cloud Run

### Managing Environment Variables (Best Practice)
For all sensitive environment variables (`DATABASE_URL`, `JWT_SECRET`, `GCS_SA_KEY_JSON_STRING`), you should store them in **Google Secret Manager**. This is the most secure method.

When deploying, you will reference these secrets instead of passing them as plain text.

### Deploy Command
Deploy the container image you pushed to your Cloud Run service.

```powershell
$FULL_IMAGE_URI = "<REGION>-docker.pkg.dev/<YOUR_GCP_PROJECT_ID>/<YOUR_REPO_NAME>/<YOUR_IMAGE_NAME>:latest"

gcloud run deploy <YOUR_CLOUD_RUN_SERVICE_NAME> \
  --image $FULL_IMAGE_URI \
  --region <YOUR_GCP_REGION> \
  --project <YOUR_GCP_PROJECT_ID> \
  --port 3000 \
  --allow-unauthenticated \
  --update-secrets=DATABASE_URL=<YOUR_DB_URL_SECRET_NAME>:latest,JWT_SECRET=<YOUR_JWT_SECRET_NAME>:latest,GCS_SA_KEY_JSON_STRING=<YOUR_GCS_KEY_SECRET_NAME>:latest \
  --set-env-vars=JWT_EXPIRES_IN=1d,GCS_BUCKET_NAME=<YOUR_GCS_BUCKET_NAME>,GCS_PROJECT_ID=<YOUR_GCP_PROJECT_ID>,FRONTEND_ORIGIN_URL=<YOUR_FRONTEND_URL>
```
*Replace all `<PLACEHOLDER>` and `YOUR_..._NAME` values with your actual resource names.*

---

## V. API Endpoints

The API is prefixed with `/api/v1`. Refer to the controllers in the source code for specific routes:

* **AuthModule**: `/auth/register`, `/auth/login`, `/auth/profile`
* **DossiersModule**: `/dossiers`
* **SubjectProfileItemsModule**: `/dossiers/:dossierId/subject-profile-items`
* **NotesModule**: `/dossiers/:dossierId/notes`
* **ArtifactsModule**: `/dossiers/:dossierId/artifacts` (with sub-routes for `generate-upload-url`, `notify-upload`, etc.)

---

## VI. Database Migrations (Production)

For production environments, `synchronize: true` in your TypeORM configuration should always be set to `false` to prevent accidental data loss. Use TypeORM migrations to manage schema changes.

1.  **Configure DataSource for TypeORM CLI**: Create `src/data-source.ts` (or similar, and adjust paths in `package.json` scripts).
2.  **Generate a Migration**: `pnpm run typeorm:generate -- src/migrations/MyNewMigration`
3.  **Run Migrations**: `pnpm run typeorm:run`
