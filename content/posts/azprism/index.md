---
date: '2025-09-05T18:28:34+02:00'
title: 'Copying Users & Groups from one Azure Application onto another'
---

# Why azprism
{{< lead >}}
**Az**ure **Pri**ncipal **S**ync **M**echanism
{{< /lead >}}

{{< github repo="pedromaca/azprism" showThumbnail=false >}}

There can be many reasons why the same users & groups need to be in multiple applications or even if you're just targeting multiple environments. In this case, `azprism` was born out of both necessity and curiosity to address specifically a SCIM Provisioning challenge, but the logic applies to any instance in need of "user & group" sync between different Service Principals.

Effectively, the goal is to copy the Users & Groups from the "original" Service Principal. However, this is not something directly possible within Azure and so it has to be done manually, leading to a waste of resources and potential mistakes. There is no "clone", "replicate" or even "export/import" button that allows you to get this same configuration into another Service Principal. Finally, some use cases which rely on SCIM do not even support "Groups of Groups", so consolidating everything into one is not an option either.

![Problem Diagram](problem.drawio.svg)

This problem becomes more complex as we have more target systems which share the same user base. Example:
- `AppABC` has users & groups provisioned into it through `AppABC-scim-provision` Service Principal
- `AppXYZ` must have the same users & groups provisioned into it but it cannot leverage the same service principal as the one used for AppABC. Hence, a new one, `AppXYZ-scim-provision` has to be created, and maintained, with the same exact configuration in terms of users and groups
- Moreover, AppXYZ has multiple environments, so the challenges multiply themselves very quickly and we end up having several `AppXYZ-scim-provision-<env>` Service Principals which must be maintained.

# Solution

```
Description:                                                                                                                                                                         
Manage principal assignments across origin/target objects

Usage:
azprism principals [command] [options]

Options:
-?, -h, --help  Show help and usage information

Commands:
add     Add missing principals from original to target
remove  Remove principals from target which are not in original
sync    Synchronize adds missing principals from original to target and removes principals from target which are not in original
```

> `azprism` was created to serve on demand needs directly from a terminal. Alternatively, Terraform `azuread_app_role_assignment` could be used to achieve the same result in an IaC manner.

The solution developed addresses the aforementioned challenges and involves the introduction of a `prism service principal` which acts as a middleman between the `original` and `target` service principals. It relies heavily on [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/use-the-api) and it adheres to the following approach:

1. GET Request against Microsoft Graph API: `https://graph.microsoft.com/v1.0/servicePrincipals/{original-object-id}/appRoleAssignedTo`
2. Identify the list of Users & Groups assigned to `original Service Principal`
3. POST Request against the Microsoft Graph API targeting the `target Service Principal` with the details previously retrieved

![Solution Diagram](solution.drawio.svg)

It is important to mention that the assignment of Users & Groups into an application means that they will get assigned an AppRole. This AppRole has a corresponding AppRoleId which can be different from one application to another even though the AppRole is the same.

This approach has some relevant pre-requisites which are listed below.

# How to leverage azprism

## Pre-requisites

1. **API Permissions** The `prism service principal` has to have, as a minimum, 'Admin Consent' granted for the following API permission
> **`Application.ReadWrite.OwnedBy`** Allows the app to create other applications, and fully manage those applications (read, update, update application secrets and delete), without a signed-in user.  It cannot update any apps that it is not an owner of.

![API Permissions](api.drawio.svg)

2. **Service Principal Ownership** The `prism service principal` has to be an owner of both ends of the Service Principals (this means the **Enterprise Application**, not the corresponding App Registration), the `original` and the `target`. Unfortunately, this ownership cannot be granted via the Azure Portal and only a user with the right permissions will be able to do it.

**Assigning ownership** of an Application (Service Principal) to another Service Principal is currently not supported via Azure Portal nor Azure CLI `az ad sp owner add` so an `az rest --method POST` request can be made as follows:

```shell
# Ensure you have the right permissions to execute these commands

# Get Object IDs
PRISM_SP_ID=$(az ad sp show --id <prism service principal app-id> --query id -o tsv) # prism service principal
ORIGINAL_SP_ID=$(az ad sp show --id <original-app-id> --query id -o tsv) # original service principal
TARGET_SP_ID=$(az ad sp show --id <target-app-id> --query id -o tsv) # target service principal

# Add prism service principal as an owner of original-app using Microsoft Graph
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$ORIGINAL_SP_ID/owners/\$ref" \
  --body "{\"@odata.id\":\"https://graph.microsoft.com/v1.0/directoryObjects/$PRISM_SP_ID\"}"

# Add prism service principal as an owner of target-app using Microsoft Graph
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$TARGET_SP_ID/owners/\$ref" \
  --body "{\"@odata.id\":\"https://graph.microsoft.com/v1.0/directoryObjects/$PRISM_SP_ID\"}"
```

Note that the Ids used in the POST requests are Object Ids of the corresponding Application through their application Id.

# Example

0. Have .NET 8 installed

1. Clone the repository
    ```shell
    git clone github.com/pedromaca/azprism
    ```

2. Set the **Environment Variables** for `azprism`:

    ```shell
    // Application Id
    export CLIENT_ID=<client-id>

    // SP secret
    export CLIENT_SECRET=<client-secret>
    
    // Tenant Id
    export TENANT_ID=<tenant-id>
    ```

3.  Run the command principals (use optional --dry-run flag to test the command without making any changes):

    ```shell
    # Sync all users and groups from original to target
    dotnet run --project src/azprism.csproj principals sync --original-id <id> --target-id <id>
    
    # Add users and groups from original missing in target
    dotnet run --project src/azprism.csproj principals add --original-id <id> --target-id <id>

    # Remove all users and groups from target which are not in original
    dotnet run --project src/azprism.csproj principals remove --original-id <id> --target-id <id>
    ```

Example output using the --dry-run flag:

```shell
info: azprism.Services.AddPrincipalsService[0]
      [DRY RUN] AzPrism will add 165 principals.
info: azprism.Services.RemovePrincipalsService[0]
      [DRY RUN] AzPrism will remove 17 principals.
```

Optionally, you can also build the project and run the executable directly:

```shell
# Publish the project
dotnet publish src/azprism.csproj -c Release -r osx-arm64 -o ~/azprism
```

From there, you can run the executable directly:

```shell
# Run the azprism executable (sync example)
./azprism principals sync --original-id <id> --target-id <id>
```

## azprism sync flowchart

{{<mermaid>}}
flowchart TD
A[Create GraphServiceClient] --> B[Execute 'azprism principals sync']
B --> C[Fetch AppRole Assignments from Original SP]
C --> D[Fetch AppRole Assignments from Target SP]
D --> E[Compare Assignments]
E --> F[Add Missing Principals to Target]
E --> G[Remove Extra Principals from Target]
F --> H[End]
G --> H
{{</mermaid>}}
