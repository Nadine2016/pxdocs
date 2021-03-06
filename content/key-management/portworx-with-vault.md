---
title: Portworx with Vault
keywords: portworx, containers, storage, vault
description: Instructions on using Vault key management with Portworx
weight: 2
disableprevnext: true
series: key-management
---

Portworx can integrate with Vault to store your encryption keys/secrets, credentials or passwords. This guide will get a Portworx cluster connected to a Vault endpoint. The vault endpoint could be used to store secrets that will be used for encrypting volumes.

### Setting up Vault {#setting-up-vault}

Peruse [this section](https://www.vaultproject.io/docs/install) for help on setting up Vault in your setup. This includes installation, setting up policies and configuring secrets.

#### Kubernetes users {#kubernetes-users}

If you are installing Portworx on Kubernetes, when generating the [Portworx Kubernetes spec file](https://install.portworx.com/):

1. Select `vault` from the “Secrets type” list
2. In the environment variables section set _Portworx Vault Environment variables_ \(see section below\) to identify Vault endpoint.

To generate Portworx spec for Kubernetes, refer instructions, [click here](/portworx-install-with-kubernetes).

If you already have a running Portworx installation, see the section on each node.

#### Other users {#other-users}

During installation,

1. Use argument `-secret_type vault -cluster_secret_key <secret-id>` when starting Portworx to specify the secret type as vault and the cluster-wide secret key.
2. Use `-e` docker option to expose the _Portworx vault environment variables_ \(see section below\).

If you already have a running Portworx installation, see the section _Adding Vault Credentials to config.json_ below.

### Portworx vault environment variables {#portworx-vault-environment-variables}

Based on your vault configuration provide the following environment variables to Portworx.

* `VAULT_ADDR=<vault-address>` **Required:** It would be used to connect to the Vault endpoint
* `VAULT_TOKEN=<vault-token>` **Required:** This token will be used for authenticating Portworx with Vault
* `VAULT_BASE_PATH=<portworx-base-path>` The base path under which portworx has access to secrets
* `VAULT_BACKEND_PATH=<custom-backend-path>` The custom backend path if different than the default `secret`
* `VAULT_CACERT=</etc/pwx/path>`
* `VAULT_CAPATH=/etc/pwx/path>`
* `VAULT_CLIENT_CERT=</etc/pwx/path>`
* `VAULT_CLIENT_KEY=/etc/pwx/path>`
* `VAULT_TLS_SERVER_NAME=<server-name>`

If Vault is configured strictly with policies then the Vault Token provided to Portworx should follow one of the following policies:

 ```text
 # Read and List capabilities on mount to determine which version of kv backend is supported
 path "sys/mounts/*"
 {
 capabilities = ["read", "list"]
 }

 # V1 backends (Using default backend)
 # Provide full access to the portworx subkey
 # Provide -> VAULT_BASE_PATH=portworx to PX (optional)
 path "secret/portworx/*"
 {
 capabilities = ["create", "read", "update", "delete", "list"]
 }

 # V1 backends (Using custom backend)
 # Provide full access to the portworx subkey
 # Provide -> VAULT_BASE_PATH=portworx to PX (optional)
 # Provide -> VAULT_BACKEND_PATH=custom-backend (required)
 path "custom-backend/portworx/*"
 {
 capabilities = ["create", "read", "update", "delete", "list"]
 }


 # V2 backends (Using default backend )
 # Provide full access to the data/portworx subkey
 # Provide -> VAULT_BASE_PATH=portworx to PX (optional)
 path "secret/data/portworx/*"
 {
 capabilities = ["create", "read", "update", "delete", "list"]
 }

 # V2 backends (Using custom backend )
 # Provide full access to the data/portworx subkey
 # Provide -> VAULT_BASE_PATH=portworx to PX (optional)
 # Provide -> VAULT_BACKEND_PATH=custom-backend (required)
 path "custom-backend/data/portworx/*"
 {
 capabilities = ["create", "read", "update", "delete", "list"]
 }
 ```

*Note: Portworx supports only the kv backend of Vault*

All the above Vault related fields as well as the cluster secret key can be set using Portworx CLI which is explained in the next section.

### Adding Vault Credentials to config.json {#adding-vault-credentials-to-configjson}

This section is relevant for either of the below 2 scenarios

* You are deploying Portworx with your configuration created before hand. Then you want to create a `/etc/pwx/config.json` before starting Portworx installation.
* You already have a working Portworx cluster so each node already has a `/etc/pwx/config.json`

Add the following `secret_type`, `cluster_secret_key` and `vault` section to the `/etc/pwx/config.json` on each node in the cluster:

```text
cat /etc/pwx/config.json
{
    "clusterid": "<cluster-id>",
    "secret": {
        "secret_type": "vault",
        "cluster_secret_key": "mysecret",
         "vault": {
             "VAULT_TOKEN": "string",
             "VAULT_ADDR": "string"
             "VAULT_CACERT": </etc/pwx/path>,
             "VAULT_CAPATH": </etc/pwx/path>,
             "VAULT_CLIENT_CERT": </etc/pwx/path>,
             "VAULT_CLIENT_KEY": </etc/pwx/path>,
             "VAULT_TLS_SERVER_NAME": <>,
        }
    }
    ...
}
```

### Key generation with Vault {#key-generation-with-vault}

The following sections describe the key generation process with Portworx and Vault which can be used for encrypting volumes. More info about encrypted volumes [here](/reference/cli/encrypted-volumes)

**Setting cluster wide secret key**

A cluster wide secret key is a common key that can be used to encrypt all your volumes. You can set the cluster secret key using the following command.

```text
/opt/pwx/bin/pxctl secrets set-cluster-key
Enter cluster wide secret key: *****
Successfully set cluster secret key!
```

This command needs to be run just once for the cluster. If you have added the cluster secret key through the config.json, the above command will overwrite it. Even on subsequent Portworx restarts, the cluster secret key in config.json will be ignored for the one set through the CLI.

**\(Optional\) Authenticating with Vault using Portworx CLI**

If you do not wish to set Vault environment variables, you can authenticate Portworx with Vault using Portworx CLI. Run the following command:

```text
/opt/pwx/bin/pxctl secrets vault login \
  --vault-address <vault-endpoint-address> \
  --vault-token <vault-token>
Successfully authenticated with Vault.
```

{{<info>}}
**Important:**  
You need to run this command on all Portworx nodes, so that you could create and mount encrypted volumes on all nodes.
{{</info>}}

{{<info>}}
**Important:**  
Make sure that the secret key has been created in Vault.
{{</info>}}

If the CLI is used to authenticate with Vault, for every restart of Portworx container it needs to be re-authenticated with Vault by running the `login` command.  
