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

### Create Cache Directories

You'll need to create at least one directory for model files, weights, and MSA databases.
The simplest way to go is to do

```
mkdir -p ~/.cache/nim
chmod -R 777 ~/.cache/nim
```

### Optional: Specify An Alternate Directory

The protein sequence databases necessary for the MSA step in AlphaFold-2 require about 500 GB
of disk space.  It can be useful to specify another file location for AlphaFold-2 that will 
get used to store these files.  For example, if you had a larger filesystem at `/scratch`, 
you might do:

```
mkdir -p /scratch/nim
chmod -R 777 /scratch/nim
```

Then you would want to set the environment variables `$ALPHAFOLD2_CACHE`, `$DIFFDOCK_CACHE`, and `$MOLMIM_CACHE` as follows using the 
example directory we just created.

```
export ALPHAFOLD2_CACHE='/scratch/nim'
export DIFFDOCK_CACHE='/scratch/nim'
export MOLMIM_CACHE='/scratch/nim'
```

### Start Up the NIMs

Starting the NIMs is as simple as running

```bash
docker compose up
```

You should see Docker pulling the containers, which can take up to several hours, depending on the speed of you internet connection.  Once the containers are pulled, they will start. It will take ~5 minutes to start the three NIMs.

### Check the Status of the NIMs

Check **AlphaFold2** status:

```bash
curl localhost:8081/v1/health/ready
```

Example output:

```bash
{"status":"ready"}
```

Check **DiffDock** status:

```bash
curl localhost:8082/v1/health/ready
```

Example output:

```bash
true
```

Check **MolMIM** status:

```
curl localhost:8083/v1/health/ready
```

Example output:

```bash
{"status":"ready"}
```

### Interacting with the NIMs

The three NIMs are available on three different ports:

- AlphaFold2: 8081
- MolMIM: 8082
- DiffDock: 8083

For a complete example, navigate to the [src](../src) directory where you'll find a Jupyter notebook with a complete generative virtual screening pipeline.
