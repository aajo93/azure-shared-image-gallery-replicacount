# Azure Shared Image Gallery (SIG) replicaCount  example
Simple example showing how to create Azure Shared Image Gallery (SIG) image with multiple replicas for higher scale deployment.

For more information about Shared Image Gallery public preview announced at Microsoft Ignite 2018 see https://azure.microsoft.com/en-us/blog/announcing-the-public-preview-of-shared-image-gallery/

## Prep steps
Copy existing managed image to one of the regions that are already supported by Shared Image Gallery (SIG)

If we still have the original VM from which the managed image was created, we can copy the managed image using Azure CLI 2.x "image-copy-extension".

```
az extension add --name image-copy-extension
az image copy --source-resource-group images-francecentral --source-object-name myimage1 --target-location westcentralus --target-resource-group images-westcentralus --cleanup --debug
```

If we do not have access to the original VM, but have the managed image snapshot, we can export the snapshot and copy it using Azure Storage Blob Copy commands.

Export environment variables to make it easier to refer to them in the commands below
```
export source_rg="images-francecentral"
export source_snapshot="mysnapshot1"

export destination_rg="images-westcentralus"
export destination_storage_account="imageswestcentralus"
export destination_location="westcentralus"
export destination_container="images"
export destination_blob="mysnapshot1.vhd"
export destination_image="myimage1"
```

Export snapshot to a SAS URL
```
export snapshot_uri=$(az snapshot grant-access --resource-group "$source_rg" --name "$source_snapshot" --duration-in-seconds 3600000 --query "accessSas" -o tsv)
```

Create destination storage account (if not yet created)
```
az storage account create -g "$destination_rg" -n "$destination_storage_account" -l "$destination_location" --sku Standard_LRS --kind StorageV2
```

Copy exported snapshot URI to the created storage account
```
export destination_storage_account_key=$(az storage account keys list -g "$destination_rg" -n "$destination_storage_account" --query "[0].value" -o tsv)
az storage container create --account-name "$destination_storage_account" --account-key "$destination_storage_account_key" --name "$destination_container"
az storage blob copy start --source-uri "$snapshot_uri" --account-name "$destination_storage_account" --destination-container "$destination_container" --destination-blob "$destination_blob" --account-key "$destination_storage_account_key"
az storage blob show --account-name "$destination_storage_account" --container "$destination_container" --name "$destination_blob" --query "properties.copy" -o json
export copied_blob_uri=$(az storage blob url --account-name "$destination_storage_account" --container "$destination_container" --name "$destination_blob" -o tsv)
```

Create managed image out of the copied VHD
```
az image create --resource-group "$destination_rg" --name "$destination_image" --os-type Linux --location "$destination_location" --source "$copied_blob_uri"
```

## Create Shared Image Gallery (SIG) Resources

Export environment variables to make it easier to refer to them in the commands below
```
export subscription_id="xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx"
export sig_rg="images-westcentralus"
export sig_name="mygallery1"
export image_name="myimage1"
export image_version="1.0.0"
export managed_image_id="/subscriptions/$subscription_id/resourceGroups/$sig_rg/providers/Microsoft.Compute/images/$image_name"
```

Create gallery
```
az group deployment create --resource-group "$sig_rg" --template-file "create-gallery.json" --parameters name="$sig_name"
```

Create shared gallery image
```
az group deployment create --resource-group "$sig_rg" --template-file "create-shared-image.json" --parameters galleryName="$sig_name" name="$image_name"
```

Create image version with replicaCount=10
```
az group deployment create --resource-group "$sig_rg" --template-file "create-version.json" --parameters galleryName="$sig_name" imageName="$image_name" name="$version_name" managedImageResourceId="$managed_image_id" replicaCount=10
```

## Explore Shared Image Gallery resources using ARM REST APIs

Get ARM REST API access_token
```
export access_token=$(az account get-access-token --query "accessToken" -o tsv)
```

Issue CURL requests to see details of the resources that were created

GET Gallery Resource
```
curl -H "Authorization: Bearer $access_token" https://management.azure.com/subscriptions/$subscription_id/resourceGroups/$sig_rg/providers/Microsoft.Compute/galleries/$sig_name?api-version=2018-06-01
```

GET Shared Image Resource
```
curl -H "Authorization: Bearer $access_token" https://management.azure.com/subscriptions/$subscription_id/resourceGroups/$sig_rg/providers/Microsoft.Compute/galleries/$sig_name/images/$image_name?api-version=2018-06-01
```

Get Shared Image Version Resource
```
curl -H "Authorization: Bearer $access_token" https://management.azure.com/subscriptions/$subscription_id/resourceGroups/$sig_rg/providers/Microsoft.Compute/galleries/$sig_name/images/$image_name/versions/$version_name?api-version=2018-06-01
```

GET Shared Image Version Replication Status
```
curl -H "Authorization: Bearer $access_token" 'https://management.azure.com/subscriptions/$subscription_id/resourceGroups/$sig_rg/providers/Microsoft.Compute/galleries/$sig_name/images/$image_name/versions/$version_name?api-version=2018-06-01&$expand=ReplicationStatus'
```
## Deploy VMs using the Shared Image Gallery

Create empty VNet with one subnet into which VMs will be deployed
```
az group create --name VNET_RG --location francecentral
az network vnet create --resource-group VNET_RG --name VNET_NAME --address-prefixes "10.0.0.0/16" --subnet-name default --subnet-prefix "10.0.0.0/24"
```

Create resource group for VM deployment
```
az group create --name sig_deployment1 --location francecentral
```

Deploy many VMs into an existing subnet using the shared image version
```
export sig_id="/subscriptions/$subscription_id/resourceGroups/$sig_rg/providers/Microsoft.Compute/galleries/$sig_name/images/$image_name/versions/$version_name"
export subnet_id="/subscriptions/$subscription_id/resourceGroups/VNET_RG/providers/Microsoft.Network/virtualNetworks/VNET_NAME/subnets/default"

az group deployment create --resource-group sig_deployment1 --template-file create-vms.json --parameters numberOfInstances=100 deploymentName=deployment1 imageId="$sig_id" subnetId="$subnet_id"
```
