<img src=https://github.com/user-attachments/assets/81d5fd14-89d3-49c7-a01a-a99e54680bf8 width="100" height="100">

# SID - Simple Integration & Deployment

<img src="https://github.com/user-attachments/assets/a1bdd6fd-1e7f-4f07-8a0b-60f947053434" width="800" alt="sid screenshot">

SID is an opinionated, (almost) no-config service to provide a very simple way to have reliable GitOps for Docker Compose and GitHub.

This project has three key objectives:
1. Provide a highly reliable way of deploying changes to `docker-compose` files from GitHub
2. Provide clear visibility on the status of each attempted deployment - whether it failed or succeeded
3. It must be as simple as possible while still achieving objective 1 and 2

### Why not Portainer or Komodo?
These apps are excellent and far more powerful than SID - however they are significantly more complicated to setup. Generally they require configuring each stack individually along with the webhook. They also have differing ability to elegantly handle mono-repo setups.
The interface of both these apps (particularly Komodo) can also be overwhelming for new users.

## Features

- 🚀 With a correctly configured `docker-compose` file for SID, and a repo structured as per below - the service is ready to go, no further setup or configuration required!
- 🪝 Provides a listener for GitHub event webhooks with signature verification 
- 💡 Context-aware deployments - the service checks to see which `docker-compose` files changed in the webhook event and only redeploys the stacks that have changed. No need for different branches or tags. 
- 🔐 Simple host validation out-of-the-box to provide basic security without needing an auth system
- 👍 A simple web interface to view activity logs, review stack status, container list and basic controls to start, stop and remove individual containers
- 📈 Basic database to capture and persist activity logs long-term
- 🐙 The container includes `git`, so this does not need to be provided on the client

## Getting Started

### Prerequisites

- Docker and Docker Compose: [Installation Guide](https://docs.docker.com/get-docker/)
- A mono-repo on GitHub containing `docker-compose.yml` inside a folder at the repo root with the name of the stack. See example below:
```
my-compose-files/ <<--- this is the repo name
├── infrastructure/
│   └── docker-compose.yml
├── media/
│   └── docker-compose.yml
└── pi-hole/
    ├── docker-compose.yml
    └── config/
        └── conf.json
```
- If your repo is **private**, a PAT token is required as an environment variable - this is explained further below in the docker config.

> [!WARNING]
> Be **very careful** with your `docker-compose` files if your repo is public, as often there are sensitive environment variables such as secret keys in plain text! Be mindful of your setup!

### Running with Docker (for end-users)

**Option 1: Using a pre-built image (recomended)**

Official images are published to GitHub Container Registry (GHCR) whenever a new [release](https://github.com/declan-wade/SID/releases) is created.

You can pull the latest image using:
```bash
docker pull ghcr.io/declan-wade/sid:latest
```
Or a specific version (e.g., `1.0.0`):
```bash
docker pull ghcr.io/declan-wade/sid:1.0.0
```
Replace `1.0.0` with the desired version tag from the releases page.

Then, when running the container, use the pulled image name (e.g., `ghcr.io/declan-wade/sid:latest` or `ghcr.io/declan-wade/sid:1.0.0`) instead of `sid-app`.
Example `docker compose` command using a pre-built image:
```docker
services:
  app:
    image: ghcr.io/declan-wade/sid:latest
    ports:
      - "3000:3000"
    environment:
      - SID_ALLOWED_HOSTS=localhost:3000
      - REPO_URL=https://<PAT>@github.com/<user>/<repo> 
      - REPO_NAME=compose-v2
      - WORKING_DIR=/home/user/sid/data
      - DB_URL=postgresql://admin:password@db:5432/sid
      - GITHUB_WEBHOOK_SECRET="abc"
    volumes:
      - ./sid/app/data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock
  db:
    image: postgres
    restart: always
    volumes:
      - ./sid/app/db:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      POSTGRES_DB: sid
```
Further information is available below on each config option.

**Option 2: Building the image locally**

1. Clone the repository:
   ```bash
   git clone https://github.com/declan-wade/SID.git
   cd SID
   ```
2. Build the Docker image:
   ```bash
   docker build -t sid-app .
   ```

## Configuration

| Name                  | Required?         | Description                                                                                                                                                                                                                                                    | Example                                                              |
|-----------------------|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| SID_ALLOWED_HOSTS     | No, for localhost only. Yes for non-localhost access to Web UI               | This is the host and port SID is running on, that you want to externally access the Web UI on. This does not affect the webhook listener If you are only accessing the Web UI on localhost, this can be assigned `localhost:3000`                                | `10.1.1.10:3000`                                                       |
| REPO_URL             | Yes               | The URL to your repo. NOTE: If your repo is private, you **must** provide a Personal Access Token (PAT) in this format: `https://<PAT>@github.com/<user>/<repo>`                                                                                                     | `https://github_pat_11AEXXXXX@github.com/john-smith/my-docker-compose` |
| REPO_NAME             | Yes               | This is the name of your repository, without the organisation or username                                                                                                                                                                                      | my-docker-compose                                                    |
| WORKING_DIR           | See description   | This is required **if** the mounts in your `docker-compose` files are using a relative path (e.g. your portainer instance has a volume bind mount like this `./portainer/data:/data`),  but this does not matter if your using a _full path_ to your mounts (e.g. `/home/user/portainer/data:/data`). **Furthermore**, the path you provide here **must** be accessible to the host or else the docker service won't be able to bring up the containers.                            |  Using the following environment variable: ` WORKING_DIR=/home/user/sid/data` with the following volume binding: `./data:/home/user/sid/data`               |
| DB_URL                | Yes               | For connecting to the postgres container. The default is fine, however you can change it if you know what your doing                                                                                                                                           | `postgresql://admin:password@db:5432/sid`                              |
| GITHUB_WEBHOOK_SECRET | If using webhooks | If your using a webhook to trigger deployments (recommended) then you must provide a secret that matches the secret provided when configuring a webhook in GitHub. GitHub does allow creation of webhooks without a secret however this will fail validation.  | `abczyx`                                                               |

Access the application by navigating to [http://localhost:3000](http://localhost:3000) (or your configured port) in your web browser.

## Using the Webhook

> [!WARNING]
> Exposing **anything** to the web poses some risk. Although SID has middleware to restrict access to the Web UI, **always** take appropriate measures to secure your endpoints, such as using a reverse proxy or zero-trust tunnel. 

The application exposes a `POST` endpoint on the `/api/webhook` route, which will need to be exposed to the web appropriately. For instance, if your service has an IP address of `10.1.1.10` and you have left the port as the default `3000`, then the route would be `10.1.1.10:3000/api/webhook`

Follow the instructions [here](https://docs.github.com/en/webhooks/using-webhooks/creating-webhooks#creating-a-repository-webhook) on how to setup webhooks on your repo. **Please note** the following:

Step 6 - use the `application/json` content type

Step 7 - specifies this is optional, however SID is expecting a secret to validate when a request is triggered, so something needs to be provided here.

Upon saving a new webhook, GitHub does a test ping request to check if the request is successful. This should come back as a success if everything is configured correctly. 

## How it works

The following diagram explains how the webhook trigger works, which is ready to run as soon as SID is brought up, no additional configuration is required. 

The "Sync GitHub" button in the UI is just for manually cloning/pulling the repo and syncing the actual structure of the compose directory with the database (this is possibly redundant, could maybe make this automatic when loading the UI)  

```mermaid
flowchart TD
    A[Webhook Trigger] --> B{Repository exists locally?}
    
    B -->|No| C[git clone repository]
    B -->|Yes| D[git pull latest changes]
    
    C --> E[Parse event payload for changed files]
    D --> E
    
    E --> F{Changed files contain docker-compose files?}
    
    F -->|No| G[End - No compose files to update]
    F -->|Yes| H[Extract directories with docker-compose changes]
    
    H --> I[For each affected directory]
    I --> J[Navigate to directory]
    J --> K[Execute: docker compose up -d --remove-orphans]
    
    K --> L{More directories?}
    L -->|Yes| I
    L -->|No| M[End - All compose files updated]
    
    K --> N{Command successful?}
    N -->|Yes| O[Log success event]
    N -->|No| P[Log error event]
    
    O --> L
    P --> L
    
```

## Development Instructions

> [!IMPORTANT]
> This repo will work with either `npm`, `pnpm` or `bun` for local development purposes, however the `Dockerfile` at build stage will be expecting a frozen `pnpm` lockfile, so ensure this has been updated with `pnpm install`

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/declan-wade/SID.git
    cd SID
    ```
2.  **Install dependencies:**
   
    ```bash
    bun install
    ```
3.  **Set up environment variables:**
    For development, `DATABASE_URL` is already configured in `prisma/schema.prisma` to use `file:./dev.db`.
    If you need to change it (e.g., to use a different database file or a server-based database), you can create a `.env` file in the root of the project and set the `DATABASE_URL` there:
    ```env
    DATABASE_URL="file:./dev.db"
    ```

4.  **Database setup:**
    The project uses Prisma for ORM and Postgres as the database.
    To initialize/reset the database and apply migrations for development:
    ```bash
    npx prisma migrate dev
    ```
    This will bootstrap a new database called `sid` with the permissions passed from the environment variables.

5.  **Run the development server:**
    ```bash
    bun dev
    ```
    Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

6.  **Running tests:**
    (Instructions for running tests - To be filled if tests are added)

## Tech Stack

- Next.js
- React
- TypeScript
- ShadCn
- Prisma
- Postgres
- Docker
- Tailwind CSS

## Acknowledgements

- Big thanks to the excellent [Homepage](https://github.com/gethomepage/homepage) project (and by extension [@shamoon](https://github.com/shamoon)) for inspiration on the `middleware` component allowing local host validation!  

## Contributing

Contributions are welcome! Please fork the repository, create a new branch for your feature or bug fix, and submit a pull request.

## License

This project is licensed under the [MIT License](https://github.com/declan-wade/SID/tree/main?tab=MIT-1-ov-file).

