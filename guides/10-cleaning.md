# Cleaning the CoAKS Environment

Delete the AKS Cluster by removing the Resource Group:

```bash
az group delete --name myCoAKSResourceGroup --yes --no-wait
```

Then delete the Users and Group in the Azure Entra:

```bash
az ad user delete --id admin@energycorp.com
az ad user delete --id alice@energycorp.com
az ad user delete --id bob@energycorp.com

az ad group delete --group myCoAKSAdminGroup
az ad group delete --group myCoAKSSolarGroup
az ad group delete --group myCoAKSEolicGroup
az ad group delete --group myCoAKSCapsuleGroup
```

Done.