### Docker Compose

In this directory you will find a Docker Compose yaml file that will start up 
the NIMs for AlphaFold-2, MolMIM, and DiffDock. Once the NIMs are started, you
will probably then want to take a look the example notebook in the `src` directory.

### Prerequisites

You will need to install:
* [Docker](https://docs.docker.com/engine/install/)
* [Docker Compose](https://docs.docker.com/compose/).
* [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

### Get Your API Key 

Navigate to the [API Catalog](https://build.nvidia.com/nvidia/generative-virtual-screening-for-drug-discovery), ensure you're logged in, and click "Download Blueprint" to generate an API Key. 

It can be convenient to save your API Key to a file for later use.  Here, we'll assume you've saved your API Key to a plain text file called `~/API_KEY`.

Set an environment variable named `NGC_CLI_API_KEY` to your API Key.

```bash
export NGC_CLI_API_KEY=$(cat ~/API_KEY)
```

### Start Up The NIMs

Starting the NIMs is as simple as running

```bash
docker compose up
```

You should see Docker pulling the containers, which can take up to several hours, depending on the speed of you internet connection.  Once the containers are pulled, they will start. It will take ~5 minutes to start the three NIMs.

### Interacting with the NIMs

The three NIMs are available on three different ports:

- AlphaFold2: 8081
- MolMIM: 8082
- DiffDock: 8083

For a complete example, navigate to the [src](../src) directory where you'll find a Jupyter notebook with a complete generative virtual screening pipeline.
