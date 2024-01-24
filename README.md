# Projected Tokens PoC

This repo contains configurations, scripts, patches, etc. used for a proof of concept to replace static infrastructure credentials with open id tokens issued by Gardener.

## Garden Landscape

The development setup uses [local gardener with extensions](https://github.com/gardener/gardener/blob/master/docs/deployment/getting_started_locally_with_extensions.md) setup. To make the local kind cluster an identity token provider, the service account issuer must be changed to the hostname that will be used to publish the OIDC discovery documents later. For this purpose, checkout [vpnachev/gardener@projected-token-poc](https://github.com/vpnachev/gardener/tree/projected-token-poc) and run

```bash
make kind-extensions-up SERVICE_ACCOUNT_ISSUER=<hostname>
```

## OIDC Discovery Charts

[charts/oidc-discovery](charts/oidc-discovery) helm chart is used to publish OIDC discovery documents in the internet.
It uses nginx as server for two static documents:

1. Open ID Configuration on  `/.well-known/openid-configuration`
1. JWKs on `/openid/v1/jwks`

The service is made available in the internet via `Ingress` resource.

:warning: The helm is designed to be deployed in Gardener Seed clusters, it might need changes to work in different environments

### How to install

```bash
$ OIDC_CONFIG=$(kubectl get --raw /.well-known/openid-configuration | base64 -w0)
$ OIDC_JWKS=$(kubectl get --raw /openid/v1/jwks | base64 -w0)
$ helm --kubeconfig ${seed_kubeconfig} upgrade \
    --install oidc-descovery . \
    --create-namespace \
    --namespace oidc-discovery \
    --set oidc.config="${OIDC_CONFIG}" \
    --set oidc.keys="${OIDC_JWKS}" \
    --set ingress.hostname="example.com"
```

## GCP Provider

### Infrastructure setup

Now, the local `kind-extensions` cluster has to be configured as Workload
Identity Provider in GCP. The following configurations are required.

1. A Workload Identity Pool
1. A Workload Identity Provider (Use the same `<hostname>` as issuer in the
   provider )
1. Create a GCP service account and grant it with `Compute Admin` and
   `Service Account Admin` permissions (those are required for the shoot
   creation).
1. Associate the service account with the workload identity provider, here the
   impersonation is configured, e.g. a k8s identity
   `system:serviceaccount:garden-local:local-poc` can impersonate the GCP
   service account from the previous step.
1. Download the "Client Library Config" from the console and save in file named
   gcp_client_config.json.
    1. For OIDC token path set `/srv/cloudprovider/token`
    1. Use `text` format type

Note, no credentials downloaded file contains no credentials, just metadata for
the project, service account, and where to expect the actual token.

### Provider GCP setup

`gardener-extension-provider-gcp` does not support out of the box projected
tokens, thus customized versions needs to be used.

1. Check out
   <https://github.com/vpnachev/gardener-extension-provider-gcp/tree/poc/managed-identity>
1. Scale down the provider-gcp controllerdeployment to 0 replicas
1. Set the `KUBECONFIG` environment variable to your seed cluster.
1. Hook your local computer to the seed cluster with

    ```bash
    ./hack/hook-me.sh <extension-provider-gcp namespace name> 8443
    ```

1. Run the provider-gcp controller in a container

    ```bash
    docker run -it --rm -p 8443:8443 \
        --workdir=/go/src/github.com/gardener/gardener-extension-provider-gcp \
        -v $PWD:/go/src/github.com/gardener/gardener-extension-provider-gcp \
        -v $KUBECONFIG:/tmp/seed.kubeconfig \
        -e KUBECONFIG=/tmp/seed.kubeconfig \
        golang:1.21

    make start \
        IGNORE_OPERATION_ANNOTATION=false \
        LEADER_ELECTION=true \
        EXTENSION_NAMESPACE=<extension-provider-gcp namespace name> \
        WEBHOOK_CONFIG_MODE=service
    ```

### Shoot setup

The Secret behind the SecretBinding still needs to set a serviceaccount.json and
this is the file downloaded from the previous step. For the purpose of the PoC,
it will be also the bearer of the identity token which is set manually.

1. Create a secret with GCP credentials

    ```bash
    kubectl create secret generic projected-token \
        --from-file=serviceaccount.json=gcp_client_config.json \
        --from-literal=token=$(kubectl -n garden-local create token local-poc --audience <aud> --duration 87600s)
    ```

1. Create a SecretBinding

    ```yaml
    apiVersion: core.gardener.cloud/v1beta1
    kind: SecretBinding
    metadata:
      name: projected-token
      namespace: garden-local
    provider:
      type: gcp
        # The type of your provider, e.g. <aws|az|gcp|alicloud|...>
    secretRef:
      name: projected-token
      namespace: garden-local
        # The name of the project prefixed with `garden-`, e.g. garden-<project-name>
    ```

1. Create your GCP shoot using the `projected-token` SecretBinding
