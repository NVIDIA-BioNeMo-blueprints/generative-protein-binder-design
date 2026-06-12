# Getting Started with Docker Compose

## Install Dependencies

To get started with docker compose, you'll need to have docker, docker compose, and the
nvidia container toolkit installed and up to date. This workflow is only compatible with 
docker compose version 2 or greater.

At a minimum, this will require:
```bash
sudo apt-get install -y docker-compose
```

## Set your personal NGC CLI API KEY Environment Variable

An NGC personal run key (with permissions scoped properly) is required
to download and run the NIMs.

```bash
export NGC_CLI_API_KEY=<YOUR NGC PERSONAL RUN KEY>
```

## Log in to nvcr.io

```bash
## Use the literal '$oauthtoken' for username and your NGC_CLI_API_KEY for the password
docker login nvcr.io --username='$oauthtoken' --password="${NGC_CLI_API_KEY}"
```



## Create your NIM cache and set its permissions

NIMs cache any model data to your local disk so that subsequent start times are faster.
You can put this cache anywhere on your system that you have permissions, but the recommended
location is `~/.cache/nim`. Note: the cache **must** have global read-write permissions so that
the NIM can write into it, so we strongly recommend against putting any files requiring higher security
in the NIM cache directory.

```bash
## Create the nim cache directory
mkdir -p ~/.cache/nim

## Make the NIM cache writable by the NIM
chmod -R 777 ~/.cache/nim

## Set an environment variable so that docker compose can find the cache
export HOST_NIM_CACHE=~/.cache/nim
```

## Start the Docker Compose Configuration

Now, you can start docker compose by running `docker compose up` from the `deploy/` directory
of the Protein Design Blueprint repository:

```bash
## From the root of the cloned Protein Design repository:
cd deploy/
docker compose up
```

This will first pull the containers then start them. When the containers start they will
pull the model for each NIM. This process can take several hours; in the case of AlphaFold2
and other large models, **expect the model download step to take from three to seven hours**
even on a fast internet connection.
