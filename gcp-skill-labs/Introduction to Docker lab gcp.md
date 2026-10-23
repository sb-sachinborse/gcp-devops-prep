
# Introduction to Docker

## Create the Target Docker Repository (Using Cloud Console)
You must create a repository before you can push any images to it. Pushing an image cannot trigger the creation of a repository, and the Cloud Build service account does not have permissions to create repositories.

1. From the **Navigation Menu** > **View all products**, under the **CI/CD** section, navigate to **Artifact Registry** > **Repositories**.
2. Click the **+ CREATE REPOSITORY** icon next to repositories.
3. Specify `my-repository` as the repository name.
4. Choose **Docker** as the format.
5. Under **Location Type**, select **Region** and choose the location: `us-central1`.
6. Click **Create**.

---

## Configure Authentication
Before you can push or pull images, configure Docker to use the Google Cloud CLI to authenticate requests to Artifact Registry.

To set up authentication to Docker repositories in the `us-central1` region, run the following command in Cloud Shell:

```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```

The command updates your Docker configuration. You can now connect with Artifact Registry in your Google Cloud project to push and pull images.

> **Note:** Alternately, you can use the gcloud CLI for a streamlined command-line approach.

### Create an Artifact Registry Repository (Using CLI)
Run the following command to create an Artifact Repository via the terminal:

```bash
gcloud artifacts repositories create my-repository \
    --repository-format=docker \
    --location=us-central1 \
    --description="Docker repository"
```

> **Note:** When you make a Google Cloud API call or use a command-line tool that requires credentials (such as the gcloud CLI or bq) with Cloud Shell for the first time, Cloud Shell prompts you with the *Authorize Cloud Shell* dialog. Click **Authorize** to allow the tool to use your credentials.

---

## Push the Container to Artifact Registry

1. Change into the directory containing your `Dockerfile`:
   ```bash
   cd ~/test
   ```

2. Run the command to build and tag the image as `node-app:0.2`:
   ```bash
   docker build -t us-central1-docker.pkg.dev/qwiklabs-gcp-03-a364938f2075/my-repository/node-app:0.2 .
   ```

3. Run the following command to check your built Docker images:
   ```bash
   docker images
   ```

**Command Output:**
```text
IMAGE                                                                             ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-world:latest                                                                f7931603f70e       20.3kB         3.96kB    U
node-app:0.1                                                                      6ab532e26f8d       1.62GB          408MB    U
node-app:0.2                                                                      663fe008d88c       1.62GB          408MB    U
us-east1-docker.pkg.dev/qwiklabs-gcp-03-738a984de075/my-repository/node-app:0.2   7126555bbfb6       1.62GB          408MB    U
```

4. Push this image to your Artifact Registry repository:
   ```bash
   docker push us-central1-docker.pkg.dev/qwiklabs-gcp-03-a364938f2075/my-repository/node-app:0.2
   ```

**Command Output:**
```text
The push refers to a repository [us-central1-docker.pkg.dev/qwiklabs-gcp-03-a364938f2075/my-repository/node-app:0.2]
057029400a4a: Pushed
342f14cb7e2b: Pushed
903087566d45: Pushed
99dac0782a63: Pushed
e6695624484e: Pushed
da59b99bbd3b: Pushed
5616a6292c16: Pushed
f3ed6cb59ab0: Pushed
654f45ecb7e3: Pushed
2c40c66f7667: Pushed
0.2: digest: sha256:25b8ebd7820515609517ec38dbca9086e1abef3750c0d2aff7f341407c743c46 size: 2419
```

5. After the push finishes, navigate back to the Cloud Console: Go to **Navigation Menu** > **View all products** > **CI/CD** > **Artifact Registry** > **Repositories**.
6. Click on **my-repository**. You should see your `node-app` Docker container created inside the repository view.

---

## Test the Image
To simulate a completely fresh production environment, clear all local containers and images instead of spinning up a separate VM.

1. Stop and remove all running local containers:
   ```bash
   docker stop $(docker ps -q)
   docker rm $(docker ps -aq)
   ```

2. Remove the local images. You must remove the child images before removing base images like `node:22-slim`:
   ```bash
   docker rmi us-central1-docker.pkg.dev/qwiklabs-gcp-03-a364938f2075/my-repository/node-app:0.2
   docker rmi node:22-slim
   docker rmi -f $(docker images -aq)
   ```

3. Verify your environment is clean:
   ```bash
   docker images
   ```

**Command Output:**
```text
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

4. Now pull and execute your clean deployment directly from Artifact Registry:
   ```bash
   docker run -p 4000:80 -d us-central1-docker.pkg.dev/qwiklabs-gcp-03-a364938f2075/my-repository/node-app:0.2
   ```

5. Test the application endpoint by routing a request to the container port mapping:
   ```bash
   curl http://localhost:4000
   ```

**Command Output:**
```text
Welcome to Cloud
```

