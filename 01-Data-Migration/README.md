# Overview
Most customers have accumulated a large collection of genomic data that will need to be moved to the cloud. It's not uncommon to have customers with at least 1PB of data on premises. In addition to this initial snapshot, they are also producing new datasets daily. This data will also need to be moved to the cloud on a periodic basis.

For the initial snapshot, there are two commonly used ways to migrate the data. We can use a storage device like [Azure Databox](https://docs.microsoft.com/en-us/azure/databox/data-box-overview) or we can move the data over the wire using the customers current connectivity to **Azure**, whether that is [Azure Express Route](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-introduction), a [Site-to-Site VPN](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/vpn) or even using the secure internet endpoints for the [Azure Storage Services](https://docs.microsoft.com/en-us/azure/storage/common/storage-introduction). 

**Azure Data Box** is not always the right, or fastest solution. It takes some time to acquire the physical Data Box, copy and load the data, ship it back, and have it loaded into your Storage Account. Depending on the amount of data, and available bandwidth, it may be faster to move the data over the wire. Make sure you review both the online, and offline options and see which one works best for your scenario.

The choices for moving the periodic datasets are more clear cut. Given the frequency and the data size of daily genomic datasets, moving the data over the wire will likely be the best option.

For this quickstart, we will focus on moving the data over the wire and demonstrate how to set up the required infrastructure. When moving the initial snapshot, we'll use [Azure AzCopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-ref-azcopy) and or the differential data we'll use {Azure Data Box Gateway](https://docs.microsoft.com/en-us/azure/databox-gateway/data-box-gateway-overview).

# Using AzCopy to move snapshot data

AzCopy is a command-line utility that you can use to copy blobs or files to, or from, a storage account. AzCopy is cross-platform, and available on Windows, Linux and MacOS. Please review the [Getting started with AzCopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10?toc=/azure/storage/blobs/toc.json) documentation to setup and start using the tool.

In order to access your storage account, you will need to choose what type of **Security Principal** you are going to use. AzCopy supports [User Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-manage-ua-identity-portal#create-a-user-assigned-managed-identity), [Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) and [Service Principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals). Because we are running the load process from outside of **Azure** <!--- Is the emphasis on the right word here? If you're going to emphasize anything I'd focus on "outside" --->, and we'll be automating the process, we'll use a **Service Principal**. We'll want to make sure the **Service Principal** has the appropriate role to read/write data to the storage account, and in this case we'll use the **Storage Blob Data Contributor** role.

## Service Principal configuration

If you don't have a **Service Principal**, follow [this guide](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals) to learn more and set one up in your environment.

Once you have one handy, there are a few minor configurations that you'll need to do before you can start using it on machine that you will use for copying data. We'll run through the set up on an Ubuntu 20.18 VM <!--- You may want to use, or even just say to use, an LTS Ubuntu distribution. -->. If you are using Windows or MacOs, please refer to the [Getting started with AzCopy](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10?toc=/azure/storage/blobs/toc.json) documentation.

There are two components of the service principal that you'll need, the **application(client-id)** and the corresponding **client-secret**.

Create an enviroment variable called **AZCOPY_SPA_CLIENT_SECRET** and set the value to your client-secret.

```console
AZCOPY_SPA_CLIENT_SECRET=*********-JZe0675*********
```

Once the secret is set, you can now use azcopy to login in as the **Service Principal**.

```console
azcopy login --service-principal  --application-id <your-client-id> --tenant-id <your-tenant-id>
```

If you were successful, you'll see a message  similar to this:


![SPN Login](./../99-Images/data-load-spn-login-sucess.png)

Now that you can log in, try running a simple command to test connectivity to your storage account. For example, this command will list the contents of the given container.

```console
azcopy list https://<account>.blob.core.windows.net/<container>
```

Great, you now have access to your **Azure Storage** account from your private network. Moving files from your private network to **Azure** can now be accomplished by running the **azcopy copy** command. Here's an example that copies the content of an entire directory.

```console
azcopy copy '/datadrive/1000-genomes' 'https://mystorageaccount.blob.core.windows.net/1000-genomes' --recursive
```

Refer to the [documentation](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-blobs-upload?toc=/azure/storage/blobs/toc.json) for proper syntax and additional examples.

Moving large files over the wire has some complexities which **AzCopy** is designed to address. Here are a few things that we've found to be helpful.

- Run the [benchmark](https://docs.microsoft.com/en-us/azure/storage/common/storage-ref-azcopy-bench) function in AzCopy. This will test connectivity between source/destination and will simulate loading some test files. The output will help you understand the expected throughput and help you right size your VM <!--I'm not sure what rightsizing a VM has to do with AzCopy, maybe the VM doesn't have enough CPU to copy files? I would explain this more clearly. -->.
- Orchestrate and monitor your jobs using  scheduler or orchestrator. **Azure Data Factory** could be used to trigger and monitor jobs. <!-- this is under the section of how to deal with complexities of AzCopy, I would move this out of here or re-word it. >
- [Check](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-configure) for **failed** jobs and [resume](https://docs.microsoft.com/en-us/azure/storage/common/storage-ref-azcopy-jobs-resume) them if needed. File uploads will fail for various reasons. Monitor your jobs and resume any jobs that fail. AzCopy resume feature will look at the job plan to figure out which files should be restarted. <!-- consider adding something about "--put-md5" or "--log-level">



# Using Azure DataBox Gateway to move periodic data

Azure Data Box Gateway is a managed file transfer solution that enables you to copy data onto it's local SMB or NFS share, and have it move that data into Azure. We are going to use **Azure Data Box Gateway** to continuously move data from your private network, to Azure. For a detailed overview and set up instructions, please review the [documentation](https://docs.microsoft.com/en-us/azure/databox-gateway/data-box-gateway-overview).

Azure Data Box Gateway is configured by building a VM on your private network that is based on an image downloaded from the Azure portal after creating the Azure Data Box resource. This [tutorial](https://docs.microsoft.com/en-us/azure/databox-gateway/data-box-gateway-deploy-prep) will walk you through the setup-by-step configuration process. The sizing requirements for the VM are discussed in the article, you will need a minimum of 8GB of RAM and 2TB of available storage for the data volume.

Once the gateway device is setup, you will create a local network share onto which you will copy data. Once data is loaded, it will be moved to the configured Storage Account on Azure.

There are few limitations of Azure Data Box Gateway highlighted [here](https://docs.microsoft.com/en-us/azure/databox-gateway/data-box-gateway-limits), most of these shouldn't be an issue for a typical genomics process.

For security best practices please review [this document](https://docs.microsoft.com/en-us/azure/databox-gateway/data-box-gateway-security). Each of the components in the solution have their own security requirements. For example, Data Box Gateway uses an activation key to connect the local VM to the Azure resources; use your own best practices for managing that key.


## Additional Resources
