# Ollama Server Setup

This guide provides instructions for running the Ollama server both locally and on Google Cloud Run.

## Local Development

Instructions for running the server on a local machine for development and testing.

### Prerequisites

1.  **Install Ollama:** [Download and install Ollama](https://ollama.com/download) for your operating system.

### Running the Server

1.  **Download Models:**
    ```sh
    ollama pull gemma3n:e2b
    ollama pull gemma3n:e4b
    ```

2.  **Start the Ollama Server:**
    This command starts the server and makes the specified model available for requests.
    ```sh
    ollama serve
    ```
    The server will be accessible at `http://127.0.0.1:11434`.

# Google Cloud Run Deployment

Instructions for deploying the Ollama server as a secure service on Google Cloud Run. Make sure to run all the following commands from the same directory as this `README.md` file.

### 1. Initial Setup

These steps configure your local `gcloud` CLI to interact with your Google Cloud project.

*   **Authenticate with Google Cloud:**
    Log in to your Google Cloud account.
    ```sh
    gcloud auth login
    ```

*   **Set the Google Cloud Project:**
    Configure `gcloud` to use your target project.
    ```sh
    gcloud config set project mlteamquiz
    ```

### 2. Create Cloud Storage Bucket

A Cloud Storage bucket is used to store the Ollama models. This bucket will be mounted as a volume in the Cloud Run service.

*   **Create the bucket:**
    ```sh
    gsutil mb -c standard -l europe-west1 gs://mlteam-ollama-models
    ```

### 3. Upload Models to Cloud Storage

The Ollama server in Cloud Run reads models from the Cloud Storage bucket. You need to upload the models from your local machine to the bucket. Make sure you have pulled the models locally first as described in the "Local Development" section.

*   **Upload the models:**
    This command copies the model files from your local Ollama store to the Cloud Storage bucket.
    ```sh
    gsutil -m cp -r ~/.ollama/models/* gs://mlteam-ollama-models/
    ```

### 4. Build and Deploy

Build the Docker image for the Ollama server and deploy it to Cloud Run.

*   **Build and Push Docker Image:**
    This command builds the Docker image using Cloud Build and pushes it to the Artifact Registry.
    ```sh
    gcloud builds submit --tag europe-west1-docker.pkg.dev/mlteamquiz/mlteam-repo/ollama_server:latest .
    ```

*   **Deploy to Cloud Run:**
    This command deploys the container image to Cloud Run as a service. The service is configured to be private (`--no-allow-unauthenticated`) and mounts the Cloud Storage bucket for model storage.
    ```sh
    gcloud beta run deploy ollamaserver \
      --image europe-west1-docker.pkg.dev/mlteamquiz/mlteam-repo/ollama_server:latest \
      --platform managed \
      --region europe-west1 \
      --no-allow-unauthenticated \
      --timeout=3600 \
      --max-instances=1 \
      --min-instances=0 \
      --memory=8Gi \
      --cpu=2 \
      --cpu-boost \
      --execution-environment=gen2 \
      --add-volume=name=model-vol,type=cloud-storage,bucket=mlteam-ollama-models \
      --add-volume-mount=volume=model-vol,mount-path=/models
    ```

### 5. Configure Permissions

Configure the necessary IAM permissions for invoking the Cloud Run service and for testing.

*   **Grant Cloud Run Invoker Role:**
    Allow the service account used by your backend service to invoke the private Cloud Run service.
    ```sh
    gcloud run services add-iam-policy-binding ollamaserver \
      --member="serviceAccount:mlteam@mlteamquiz.iam.gserviceaccount.com" \
      --role=roles/run.invoker \
      --platform=managed \
      --region=europe-west1
    ```

*   **Allow User to Impersonate Service Account:**
    To test the service from your local machine, you need to generate an identity token. This requires your user account to have permission to act as the service account.
    ```sh
    gcloud iam service-accounts add-iam-policy-binding mlteam@mlteamquiz.iam.gserviceaccount.com \
      --member="user:$(gcloud config get-value account)" \
      --role="roles/iam.serviceAccountTokenCreator"
    ```

### 6. Testing the Deployed Service

Here’s how to test your newly deployed Ollama server.

*   **Test with Authentication:**
    This is the standard way to interact with your secure Cloud Run service.

    1.  **Generate an Identity Token:**
        This command creates a temporary identity token by impersonating the service account. This token proves the request is authorized.
        ```sh
        export ID_TOKEN=$(gcloud auth print-identity-token --impersonate-service-account="mlteam@mlteamquiz.iam.gserviceaccount.com" --audiences="https://ollamaserver-372609419130.europe-west1.run.app")
        ```

    2.  **Call the Service with the Token:**
        Use the generated token in the `Authorization` header to make a request.
        ```sh
        curl -X POST "https://ollamaserver-372609419130.europe-west1.run.app/api/generate" \
          -H "Authorization: Bearer $ID_TOKEN" \
          -H "Content-Type: application/json" \
          -d '{
            "model": "gemma3n:e4b",
            "prompt": "Why is the sky blue?"
          }'
        ```

*   **Test Locally via Cloud Run Proxy (No Authentication):**
    The proxy command forwards a local port to the Cloud Run service, handling authentication automatically. This is useful for quick, local testing.

    1.  **Start the Proxy:**
        This command will occupy the current terminal.
        ```sh
        gcloud run services proxy ollamaserver --port=9090
        ```

    2.  **Call the Local Proxy:**
        Open a new terminal and run the following command. You can now send requests to `localhost:9090` without an authentication header.
        ```sh
        curl http://localhost:9090/api/generate -d '{
          "model": "gemma3n:e4b",
          "prompt": "Why is the sky blue?"
        }'
        ```