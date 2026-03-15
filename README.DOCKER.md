# Offline Install Instructions: Docker

These instructions are for setting up a persistent instance of the Homebrewery application locally using Docker.

If you intend to **develop** with Homebrewery, see the [Development Containers](#development-containers) section below instead of the production install steps. Using Docker to deploy MongoDB locally for development is also a great approach; a ready-made dev Compose file is provided.

# Install Docker

## Docker Desktop (MacOS/Windows)

Windows and Mac installs use Docker Desktop. Current install instructions are below.

* [Mac](https://docs.docker.com/desktop/mac/install/)
* [Windows](https://docs.docker.com/desktop/windows/install/)

You can set up the docker engine to start on boot via the Docker desktop UI.

## Docker Engine

Linux installs use Docker Engine. Docker provides installers and instructions for several of the most common distrubutions. If you do not see yours listed, it is very likely supported indirectly by your distribution.

* [Arch](https://docs.docker.com/desktop/setup/install/linux/archlinux/)
* [CentOS](https://docs.docker.com/engine/install/centos/)
* [Debian](https://docs.docker.com/engine/install/debian/)
* [Fedora](https://docs.docker.com/engine/install/fedora/)
* [RHEL](https://docs.docker.com/engine/install/rhel/)
* [Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

### Post installation steps
[Manage Docker as a non-root user (highly recommended)](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user) 
[Enable Docker to start on boot (highly recommended)](https://docs.docker.com/engine/install/linux-postinstall/#configure-docker-to-start-on-boot)

# Build Homebrewery Image

Next we build the homebrewery docker image. Start by cloning the repository. 

```shell
git clone https://github.com/naturalcrit/homebrewery.git
cd homebrewery
```

Make an changes you need to `config/docker.json` then build the image. If it does not exist,the below as a template.

```
{
"host" : "localhost:8000",
"naturalcrit_url" : "local.naturalcrit.com:8010",
"secret" : "secret",
"web_port" : 8000,
"mongodb_uri": "mongodb://172.17.0.2/homebrewery",
}
```

```shell
docker-compose build homebrewery
```

# Add Mongo container

Once docker is installed and running, it is time to set up the containers. First up, Mongo.

```shell
docker run --name homebrewery-mongodb -d --restart unless-stopped -v mongodata:/data/db -p 27017:27017 mongo:latest
```

Older CPUs may run into an issue with AVX support. 
```
WARNING: MongoDB 5.0+ requires a CPU with AVX support, and your current system does not appear to have that!
 see https://jira.mongodb.org/browse/SERVER-54407
 see also https://www.mongodb.com/community/forums/t/mongodb-5-0-cpu-intel-g4650-compatibility/116610/2
 see also https://github.com/docker-library/mongo/issues/485#issuecomment-891991814
```
If you see a message similar to this, try using the bitnami mongo instead.

```shell
docker run --name homebrewery-mongodb -d --restart unless-stopped -v mongodata:/data/db -p 27017:27017 bitnami/mongo:latest
```

If your distribution is running on an arm device such as a Raspberry Pi, you will need to run the arm-built MongoDB v4.4.

```shell
docker run --name homebrewery-mongodb -d --restart unless-stopped -v mongodata:/data/db -p 27017:27017 arm64v8/mongo:4.4
```

## Run the Homebrewery Image
```shell
# Make sure you run this in the homebrewery directory
docker run --name homebrewery-app -d --restart unless-stopped -e NODE_ENV=docker -v $(pwd)/config/docker.json:/usr/src/app/config/docker.json -p 8000:8000 docker.io/library/homebrewery:latest
```

**NOTE:** If you are running from the Windows command line, this will not work as `$(pwd)` is not valid syntax. Use this command instead:
```shell
# Make sure you run this in the homebrewery directory
docker run --name homebrewery-app -d --restart unless-stopped -e NODE_ENV=docker -v %cd%/config/docker.json:/usr/src/app/config/docker.json -p 8000:8000 docker.io/library/homebrewery:latest
```


## Updating the Image

When Homebrewery code updates, your docker container will not automatically follow the changes. To do so you will need to rebuild your homebrewery image. 

First, return to your homebrewery clone (from Build Homebrewery Image above) or recreate the clone if you deleted your copy of the code. 

First, delete the existing image.

```shell
docker rm -f homebrewery-app
```

Next, update the clone's code to the latest version.

```shell
cd homebrewery
git checkout master
git pull upstream master
```

Finally, rebuild and restart the homebrewery image.

```shell
docker-compose build homebrewery
docker run --name homebrewery-app -d --restart unless-stopped -e NODE_ENV=docker -v $(pwd)/config/docker.json:/usr/src/app/config/docker.json -p 8000:8000 docker.io/library/homebrewery:latest
```

**NOTE:** If you are running from the Windows command line, this will not work as `$(pwd)` is not valid syntax. Use this command instead:
```shell
# Make sure you run this in the homebrewery directory
docker run --name homebrewery-app -d --restart unless-stopped -e NODE_ENV=docker -v %cd%/config/docker.json:/usr/src/app/config/docker.json -p 8000:8000 docker.io/library/homebrewery:latest
```

---

# Development Containers

Two approaches are provided for running Homebrewery inside Docker during active development. Both mount your source tree into the container so that edits are immediately reflected without a full image rebuild. The server boots with `NODE_ENV=local`, which enables Vite in middleware mode (Hot Module Replacement).

## Option A — VS Code Dev Container / GitHub Codespaces

The `.devcontainer/` folder contains the full configuration required by [VS Code Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) and [GitHub Codespaces](https://github.com/features/codespaces).

### Prerequisites
- [VS Code](https://code.visualstudio.com/) with the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) installed, **or** a GitHub Codespaces account.
- Docker Desktop / Docker Engine running locally (VS Code only).

### Usage

**VS Code:**
1. Open the repository folder in VS Code.
2. When prompted *"Reopen in Container"*, click **Reopen in Container**.  
   Alternatively, open the Command Palette (`Ctrl+Shift+P`) and run **Dev Containers: Reopen in Container**.
3. VS Code rebuilds the container, installs Node dependencies (`npm install`), and attaches.
4. In the integrated terminal, start the application:
   ```shell
   npm start
   ```
5. VS Code forwards port `8000` automatically. Open the **Ports** panel or click the link in the terminal to view the running app.

**GitHub Codespaces:**
1. Click **Code → Codespaces → Create codespace on this branch** in the GitHub UI.
2. Once the codespace is ready, run `npm start` in the terminal.
3. Codespaces makes port `8000` available via a forwarded HTTPS URL shown in the **Ports** panel.

### What the dev container provides
- Node 22 (Alpine) runtime matching the production image.
- MongoDB service reachable at `mongodb://mongodb:27017/homebrewery`.
- VS Code extensions: ESLint, Prettier, MongoDB for VS Code, JS Debugger, Code Spell Checker.
- Source tree bind-mounted at `/workspace`.

---

## Option B — Standalone Development Compose

If you prefer to work in your local editor without VS Code's Dev Containers feature, `docker-compose.dev.yml` provides an equivalent environment.

```shell
# Start MongoDB + the Homebrewery dev server (Vite HMR enabled)
docker compose -f docker-compose.dev.yml up
```

The first start may take a minute while `npm install` runs inside the container. Subsequent starts reuse the cached `node_modules` volume and are much faster.

Open [http://localhost:8000](http://localhost:8000) in your browser.

### Stopping the dev environment

```shell
docker compose -f docker-compose.dev.yml down
```

To also remove the persistent MongoDB data volume:

```shell
docker compose -f docker-compose.dev.yml down -v
```


