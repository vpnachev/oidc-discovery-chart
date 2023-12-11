# Projected Tokens PoC

This repo contains configurations, scripts, patches, etc. used for a proof of concept to replace static infrastructure credentials with open id tokens issued by Gardener.

## Garden Landscape

The development setup uses [local gardener with extensions](https://github.com/gardener/gardener/blob/master/docs/deployment/getting_started_locally_with_extensions.md) setup. To make the local kind cluster an identity token provider, the service account issuer must be changed to the hostname that will be used to publish the OIDC discovery documents later. For this purpose, checkout [vpnachev/gardener@projected-token-poc](https://github.com/vpnachev/gardener/tree/projected-token-poc) and run

```bash
$ make kind-extensions-up SERVICE_ACCOUNT_ISSUER=<hostname>
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