---
title: Portworx with DC/OS Secrets
keywords: Portworx, containers, storage, dcos
description: Instructions on using DCOS secrets key management with Portworx
weight: 4
disableprevnext: true
series: key-management
---

Portworx can integrate with DC/OS Secrets to store your encryption keys/secrets and credentials. This guide will help you configure Portworx to connect to DC/OS Secrets. DC/OS Secrets can then be used to store Portworx secrets for Volume Encryption and Cloud Credentials.

{{<info>}}
**Note:**  
Secrets is an DC/OS Enterprise only feature
{{</info>}}

{{<info>}}
**Note:**  
Supported from PX Enterprise 1.4 onwards
{{</info>}}

### Configuring DC/OS Secrets with Portworx {#configuring-dcos-secrets-with-portworx}

**Configuring permissions for Secrets**

To access secrets, Portworx needs credentials of a user. This user should have permissions to access the secrets under a base secrets path. For instance, you can grant permissions to a user to access secrets under `pwx/secrets` base path, using DC/OS enterprise cli:

```text
dcos security org users grant <username> dcos:secrets:default:pwx/secrets/* full
```

**Enabling Secrets in Portworx**

During installation or when updating an existing Portworx framework, enable the feature from Secrets section.

![portworx-dcos-secret](/img/dcos-portworx-secrets-setup.png)

The `base path` is the secrets path under which Portworx will read/write secrets. If not specified, Portworx will look for secrets at the top level.

The `dcos username secret` and `dcos password secret` are the paths to secrets, where Portworx will look for credentials of the user to access the secrets. This user should have full access to secrets under the `base path`.

If you want only Portworx framework to access the username and password secrets path, the path should have prefix same as Portworx service name \(default service name is `portworx`\).

**Update config.json for existing installation**

If the Portworx framework is already installed, you will need to update the `/etc/pwx/config.json` on all nodes to start using DC/OS secrets by default. You still need to edit the framework from the above section, so that you don’t have to update the _config.json_ for new nodes.

Add the following `secret_type` and `cluster_secret_key` fields in the `secret` section to the `/etc/pwx/config.json` on each node in the cluster:

```text
{
    "clusterid": "",
    "secret": {
        "secret_type": "dcos",
        "cluster_secret_key": "pwx/secrets/cluster-wide-secret-key"
    },
    ...
}
```

You need to restart Portworx for the _config.json_ to take effect:

```text
sudo systemctl restart portworx
```

### Key generation with DC/OS {#key-generation-with-dcos}

The following sections describe the key generation process with Portworx and DC/OS which can be used for encrypting volumes. For more information about encrypted volumes, [click here](/reference/cli/encrypted-volumes).

**Setting cluster wide secret key**

Create a secret in DC/OS using the enterprise cli:

```text
dcos security secrets create --value=<secret-value> pwx/secrets/cluster-wide-secret-key
```

For more details on ways to create Secrets in DC/OS refer [DC/OS documentaion](https://docs.mesosphere.com/1.11/security/ent/secrets/create-secrets)

A cluster wide secret key is a common key that can be used to encrypt all your volumes. You can set the cluster secret key using the following command:

```text
/opt/pwx/bin/pxctl secrets set-cluster-key \
  --secret pwx/secrets/cluster-wide-secret-key
Successfully set cluster secret key
```

This command needs to be run just once for the cluster. If you have added the cluster secret through the config.json, the above command will overwrite it. Even on subsequent Portworx restarts, the cluster secret key in _config.json_ will be ignored for the one set through the CLI.

**\(Optional\) Authenticating with DC/OS Secrets using Portworx cli**

If you do not wish to pass the DC/OS credentials through the framework, you can authenticate Portworx with DC/OS Secrets using Portworx cli. Run the following command:

```text
/opt/pwx/bin/pxctl secrets dcos login \
  --username <dcos-username> \
  --password <dcos-password> \
  --base-path <optional-base-path>
Successfully authenticated with DC/OS Secrets.
** WARNING, this is probably not what you want to do. This login will not be persisted across PX or node reboots and also expire in 5 days. Please provide your login information through package config or refer docs.portworx.com for more information.
```
{{<info>}}
**Important:**  
You need to run this command on all Portworx nodes, so that you could create and mount encrypted volumes on all nodes.
{{</info>}}

If the cli is used to authenticate with DC/OS Secrets, for every restart of Portworx container it needs to be re-authenticated with DC/OS Secrets by running the `login` command.
