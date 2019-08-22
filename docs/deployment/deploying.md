# Deploying A Dragonchain

In order to deploy a Dragonchain, helm is required.

## Create Chain Secrets

Each chain has a list of secrets that it requires.
These secrets need to be generated and saved to kubernetes before a chain
can be deployed.

Chain secrets can be generated and deployed with the following commands:

(Note you will need xxd (often packaged with vim) and openssl for these
commands to work; If you are not running linux, you can easily do this in a
docker container such as `ubuntu:latest` and simply ensure that you run
`apt update && apt install -y openssl xxd` in order to have the requirements
to generate the private keys)

```sh
BASE_64_PRIVATE_KEY=$(openssl ecparam -genkey -name secp256k1 | openssl ec -outform DER | tail -c +8 | head -c 32 | xxd -p -c 32 | xxd -r -p | base64)
HMAC_ID=$(tr -dc 'A-Z' < /dev/urandom | fold -w 12 | head -n 1)
HMAC_KEY=$(tr -dc 'A-Za-z0-9' < /dev/urandom | fold -w 43 | head -n 1)
SECRETS_AS_JSON="{\"private-key\":\"$BASE_64_PRIVATE_KEY\",\"hmac-id\":\"$HMAC_ID\",\"hmac-key\":\"$HMAC_KEY\",\"registry-password\":\"\"}"
kubectl create secret generic -n dragonchain "d-INTERNAL_ID-secrets" --from-literal=SecretString="$SECRETS_AS_JSON"
# Note INTERNAL_ID from the secret name should be replaced with the value of .global.environment.INTERNAL_ID from the helm chart values (opensource-config.yaml)
```

## Helm Chart

**Please Note**: The helm chart is subject to significant changes.
A helm chart is provided here, but will have better support in the
future.

Both the helm chart and a template for the necessary values can be downloaded
[HERE](links).

### Deploying Helm Chart

Before deploying the helm chart, a few variables need to be set in the
`opensource-config.yaml` file. This file is mostly self-documenting, so see the
comments for which values must be overridden.

Once the values are set, install the helm chart with:

```sh
helm install dragonchain-k8s-0.9.0.tgz -f opensource-config.yaml --name my-dragonchain --namespace dragonchain
```

## Checking Deployment

If the helm chart deployed successfully, there should now be pods for your
new chain in the `dragonchain` kubernetes namespace. You can check by running
`kubectl get pods -n dragonchain`.

### Get The Chain ID

In order to get the chain's ID required for use in the SDK(s) or cli, run the
following command:

(Replace <POD_NAME_HERE> with one of the core image chain pods. Eg:
`d-e2603f14-1a3d-4a47-9dce-ab0eba579850-tx-processor-5f49d9k5vpn`)

```sh
kubectl exec -n dragonchain <POD_NAME_HERE> -- python3 -c "from dragonchain.lib.keys import get_public_id; print(get_public_id())"
```

### Using The SDK

With the chain deployed, the SDK(s) (or CLI tool) can be configured with the
`HMAC_ID` and `HMAC_KEY` from earlier (when creating the secrets), as well as
the chain ID from above, and finally an endpoint into your webserver's
kubernetes service.

This endpoint can be proxied without any sort of ingress by using the
`kubectl proxy` command with the webserver pod for port 8080, and using
`http://localhost:8080` for the chain endpoint.

If you have configured the chain to be accessible to Dragon Net, then you can
use its public `DRAGONCHAIN_ENDPOINT` value as the endpoint for your chain
without tunneling anything through kubernetes. See the section below for more
details on exposing your chain to the internet.

When using all these pieces of information with the SDK or CLI, they should be
able to interact with the Dragonchain as intended.

### Checking Dragon Net Configuration

In order for your chain to be working with dragon net, a few things need to be
confirmed. Firstly, your chain must have been correctly registered with dragon
net. In order to check this, you can check the public matchmaking service for
your chain's registration. In order to do this, use your public chain id from
earlier, and run the following:

```sh
curl https://matchmaking.api.dragonchain.com/registration/PUBLIC_CHAIN_ID_HERE
```

If you see a registration returned, then your chain has successfully
registered. If there is no registration, then for some reason your chain has
failed to register with matchmaking, and you should consult transaction
processor logs for more details.

If your chain is registered with the matchmaking service, the only other
requirement is that your chain is accessible from the greater internet via the
URL that the chain registered with dragon net.

In order to check this quickly, simply run the following on a computer that is
connected to the internet, but not running the chain and run the following
command (requires `curl` and `jq`):

```sh
curl "$(curl https://matchmaking.api.dragonchain.com/registration/PUBLIC_CHAIN_ID_HERE -s | jq -r .url)"/health
```

If that command returns OK, then your chain is registered and connectable! As
long as your pods are not crashing, your chain should be able to work with
Dragon Net.

Here is a list of things that could be wrong depending on the various failure
outputs of the command above:

| Error                                          | Problem                                                                                                                                                                                                            |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `Could not resolve host: null`                 | Chain is not registered correctly with matchmaking. Confirm that the transaction processor pod is not crashing, and the `DRAGONCHAIN_ENDPOINT` variable is set correctly in `opensource-config.yaml`               |
| `Could not resolve host: ...`                  | Chain's endpoint is not set correctly. Ensure that the `DRAGONCHAIN_ENDPOINT` variable is set correctly in `opensource-config.yaml` and that the DNS name in that endpoint resolves to where your chain is running |
| `Failed to connect to ...: Connection refused` | The chain is not properly exposed to the internet. See below for more details                                                                                                                                      |

#### Ensuring Chain is Reachable From The Internet

In order to use Dragon Net, your chain must be properly exposed to the
internet. By default, the opensource config values will configure the chain
to be exposed as a nodeport service on port 30000.

This means that the kubernetes cluster must be exposed to the internet on port
30000, and the `DRAGONCHAIN_ENDPOINT` value in the `opensource-config.yaml`
must be set to point to this location. This can either be a configured DNS
record, or a raw ip address. I.e. `http://1.2.3.4:30000` or
`http://abc.com:30000` if you've configured the DNS record abc.com to point to
the correct ip address of your cluster.

If you are using minikube, 2 things have to happen to be able to hit this
nodeport service from the greater internet:

1. If behind a NAT, port 30000 will have to be port-forwarded to the computer
   running minikube. Port forwarding is outside of the scope of this
   documentation, however various guides can be found online depending on your
   particular router.

2. If using minikube in a VM (which is the default unless you're running
   minikube on linux with `--vm-driver=none`), then your host computer must
   forward traffic it receives on port 30000 to the minikube VM. The process
   for setting this up is different depending on the vm driver that you are
   using. If you are using the default virtualbox vm driver, you can follow
   [these steps](https://cwienczek.com/2017/09/reaching-minikube-from-other-devices/).

In order to check that your chain is exposed correctly, simply run the curl
command from the section above.