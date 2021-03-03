# Overview

[Nextflow](https://github.com/nextflow-io/nextflow) is a bioinformatics workflow manager that enables the development of portable and reproducible workflows. Using Nextflow, you can deploy workflows on a variety of execution platforms including local, Kubernetes clusters and also on HPC. For this quickstart we will deploy our workflows to Azure Batch.

# Nextflow on Azure..

Nextflow on Azure has very few moving parts making it relatively easy to get started. Nextflow will run on any POSIX compatible system, we'll use an Ubuntu 18.04 running on Azure. You can also run this on a Windows 10 machine, using the [WSL](https://docs.microsoft.com/en-us/windows/wsl/about). Nextflow also requires Bash 3.2 (or later) and Java 8 (or later, up to 15).

We are going to use Azure Batch as the executor. For the execution in a cluster of computers the use a shared file system is required to allow the sharing of tasks input/output files.

At the time of this writing, Azure support is still in public preview. The installation instructions will be slightly different once it's generally available. We'll reference documentation found in [this install guide](https://www.nextflow.io/docs/edge/index.html).

## GA Install Instructions:

```shell
curl -s https://get.nextflow.io | bash
```

## Edge-Release (Public Preview) Install Instructions

The support for Azure Cloud requires Nextflow version 21.02.0-edge or later. If you don’t have it installed use the following command to download it in your computer:

```shell
export NXF_EDGE=1
curl get.nextflow.io | bash
```

After running above, command check the version of Nextflow installed by running this command.

```shell
nextflow -v
```

If the version is not the current release version, it wasn's for my install, then you'll have one additional install method.
- Go to the [Nextflow releases page](https://github.com/nextflow-io/nextflow/releases) on Github.
- Under the **Assets** header, get the nextflow.{version}-edge-all url.
- On a terminal window, running the following command with the asset URL.
  - ```shell
    wget -qO- ASSET-URL-FROM-ABOVE | bash
    ```
This should install the latest release version of Nextflow.

## Running Nextflow on Azure

Before we run any workflows on Azure, let's run through a [simple example](https://www.nextflow.io/docs/edge/getstarted.html#your-first-script) running locallly to make sure everything's working as intended. This is also a good introduction to Nextflow.

