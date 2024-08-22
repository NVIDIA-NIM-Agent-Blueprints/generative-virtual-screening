# Generative virtual screening

## Getting started

### Detailed Documentation

Please see the detailed documentation [here](https://nim-tme.gitlab-master-pages.nvidia.com/-/documentation/-/jobs/107747773/artifacts/_build/docs/bionemo/caddvs/latest/overview.html) for a detail guide on getting started with the generative virtual screening workflow.

### Setting up with Docker Compose

1. You will need to install docker, the NVIDIA container toolkit, and docker compose.
2. Then, you can run the NIM workflow server using the following command:

```bash
cd docker-compose
docker compose up
```

This will take ~5 minutes to start the three NIMs.

### Interacting with the NIMs

The three NIMs are available on three different ports:

- AlphaFold2: 8081
- MolMIM: 8082
- DiffDock: 8083

You can hit the endpoints for each NIM behind their respective port.

## Notebook

An example of how to call each generative virtual screening step is located in `nbs/generative-virtual-screening.ipynb`.
