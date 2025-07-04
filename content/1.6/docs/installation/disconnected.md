---
title: "Disconnected Environment"
date: 2025-07-01
Weight: 10
---
To install the Orchestrator and its required components in a disconnected environment, there is a need to mirror images and NPM packages.
Please ensure the images are added using either `ImageDigestMirrorSet` or `ImageTagMirrorSet`, depending on the format of their values.

## Images for a disconnected environment
The following images need to be added to the image registry:

> **Recommendation:**  
> When fetching the list of required images, ensure that you are using the latest version of the bundle operator when appropriate. This helps avoid missing or outdated image references.

### RHDH Operator:
```
registry.redhat.io/rhdh/rhdh-hub-rhel9@sha256:8729c21dc4b6e1339ed29bf87e2e2054c8802f401a029ebb1f397408f3656664
registry.redhat.io/rhdh/rhdh-operator-bundle@sha256:f2d99c68895d8e99cfd132c78bc39be5f2d860737f6e7d2520167404880ed865
registry.redhat.io/rhdh/rhdh-rhel9-operator@sha256:2f72c8706af43c0fbf8afc82d1925c77887aa7c3c3b1cb28f698bc4e4241ed4d
registry.redhat.io/rhel9/postgresql-15@sha256:ddf4827c9093a0ec93b5b4f4fd31b009c7811c38a406187400ab448579036c6c
```

### OpenShift Serverless Operator:
```
registry.access.redhat.com/ubi8/nodejs-20-minimal@sha256:a2a7e399aaf09a48c28f40820da16709b62aee6f2bc703116b9345fab5830861
registry.access.redhat.com/ubi8/openjdk-21@sha256:441897a1f691c7d4b3a67bb3e0fea83e18352214264cb383fd057bbbd5ed863c
registry.access.redhat.com/ubi8/python-39@sha256:27e795fd6b1b77de70d1dc73a65e4c790650748a9cfda138fdbd194b3d6eea3d
registry.redhat.io/openshift-serverless-1/kn-backstage-plugins-eventmesh-rhel8@sha256:69b70200170a2d399ce143dca9aff5fede2d37a74040dc5ddf2206deadc9a33f
registry.redhat.io/openshift-serverless-1/kn-client-cli-artifacts-rhel8@sha256:d8e04e8d46ecec005504652b8cb4ead29452a6a89e47d568df0a24971240e9d9
registry.redhat.io/openshift-serverless-1/kn-client-kn-rhel8@sha256:989cb97cf626ae8637b32d519802250d208f466a5d6ff05d6bab105b978c976a
registry.redhat.io/openshift-serverless-1/kn-ekb-dispatcher-rhel8@sha256:4cb73eedb5c7841bff08ba5e55a48fde37ed9a0921fb88b381eaa7422fe2b00d
registry.redhat.io/openshift-serverless-1/kn-ekb-kafka-controller-rhel8@sha256:4fa519b1d4ef7f0219bae21febe73012ca261c12b3c08a9732088b7dfe37f65a
registry.redhat.io/openshift-serverless-1/kn-ekb-post-install-rhel8@sha256:402956ddf4f8da30aa234cf1d151b02f1bef29de604cad2441d65584117a3912
registry.redhat.io/openshift-serverless-1/kn-ekb-receiver-rhel8@sha256:bd48166615c132dd95a3792a6c610b1d977bad7c126a5532c47330ad3899e1ef
registry.redhat.io/openshift-serverless-1/kn-ekb-webhook-kafka-rhel8@sha256:7a4ffa3ae32dc289917b9a9c7c5ca251dc8586ba64719a126164656eecfeef14
registry.redhat.io/openshift-serverless-1/kn-eventing-apiserver-receive-adapter-rhel8@sha256:8ebbf3cd6a980896e03dc4818dede80856743c24a551d9c399f9b65c0816e2b3
registry.redhat.io/openshift-serverless-1/kn-eventing-channel-controller-rhel8@sha256:b3c9b5db3db34f454a86a81b87843934a5b8e5960cf1fa446650a35b7c2b1778
registry.redhat.io/openshift-serverless-1/kn-eventing-channel-dispatcher-rhel8@sha256:97adc8d4ab32770e00a2ae0096d45d9cd0c053a99292202bc24e6e9a60d92970
registry.redhat.io/openshift-serverless-1/kn-eventing-controller-rhel8@sha256:d6aff2e731bd8fa4f8a472ab2b6cb08103e0ba04ba353918484813864d89c082
registry.redhat.io/openshift-serverless-1/kn-eventing-filter-rhel8@sha256:e348715064edc914fd45071cb2e5e0e967bd26ce0542372a833a4ede78bf2822
registry.redhat.io/openshift-serverless-1/kn-eventing-ingress-rhel8@sha256:4519eba6fa2a6c6c10f0d97992c1e911ea1ce4cf00ac9025b9b334671b0d1e14
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-aws-ddb-streams-source-rhel8@sha256:6e2272266a877c42350c6e92bd9d97e407160de8bc29c1ab472786409548f69d
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-aws-s3-sink-rhel8@sha256:a6649ecd10ea7e3cca8d254a4a4a203d585cf1a485532fcb8f77053422ab0405
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-aws-s3-source-rhel8@sha256:ac8fad706d8e47118572a5c99f669b337962920498fd4c31796e2e707f8ff11e
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-aws-sns-sink-rhel8@sha256:e0b8f3759beb0a01314c3e6f9a165d286ac7e0e5ed9533df30209f873d3e8787
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-aws-sqs-sink-rhel8@sha256:7fc8171b21af336f5c512d0f484e363d0d32f6f11211621f572827cf71bf4cf6
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-aws-sqs-source-rhel8@sha256:925b30dbcc13075348fa35ad8e28abad88b1e632e45ff76bcd40dcacf1eaf5c1
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-log-sink-rhel8@sha256:c4641ac936196229a6dc035194799d24493eaa45cc3e0b21d79a9704860d2028
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-timer-source-rhel8@sha256:3c054f0fbbeb1428b8d88927d6b219bf5ba8c744434ebc4013351ad6494540a3
registry.redhat.io/openshift-serverless-1/kn-eventing-integrations-transform-jsonata-rhel8@sha256:1451bcf5004a32a6a183836ebf3f5c0af397da6c8d176a36bcc750c726e1f408
registry.redhat.io/openshift-serverless-1/kn-eventing-istio-controller-rhel8@sha256:a39bc62f77a5303f286e43bc8c47bb0452ad6f44228efc3e8d54798b5aaeb4d6
registry.redhat.io/openshift-serverless-1/kn-eventing-jobsink-rhel8@sha256:2553b7302376ec89216934b783e9db8122693f74b428a41e94c5ec7ffc48a414
registry.redhat.io/openshift-serverless-1/kn-eventing-migrate-rhel8@sha256:6538bbb2a59b31e03d2e74e93db81b15647308812f2354d6868680d8b48a706c
registry.redhat.io/openshift-serverless-1/kn-eventing-mtchannel-broker-rhel8@sha256:65c7c98a65f09ff01ef875d505be153bad54213bf6c3210fecee238e45887b0b
registry.redhat.io/openshift-serverless-1/kn-eventing-mtping-rhel8@sha256:887f33ae9c7d8e52764b3af4a78898769cd52eb47e6e9913fe71d7e890d9816a
registry.redhat.io/openshift-serverless-1/kn-eventing-webhook-rhel8@sha256:4a2924e282a3612e00de4bfee5a8c963c9b65b962a4c7d72f999bd493026f92a
registry.redhat.io/openshift-serverless-1/kn-plugin-event-sender-rhel8@sha256:f7795088777ea84fc6180b81b6131962944e34918e2c06671033a1a572581773
registry.redhat.io/openshift-serverless-1/kn-plugin-func-func-util-rhel8@sha256:b0eb1f0b2f180afb207186267601665f2979c4cf21a0e434e7601123e3826716
registry.redhat.io/openshift-serverless-1/kn-serving-activator-rhel8@sha256:4cf5431ee984d7cb7e6a87504e151a31130e18f1448d1eca56fbc294ee3020e4
registry.redhat.io/openshift-serverless-1/kn-serving-autoscaler-hpa-rhel8@sha256:f55ccbe4baf5829f98eb4fe7f802165d9209fe34dc8854a4eef70e471dcc1f97
registry.redhat.io/openshift-serverless-1/kn-serving-autoscaler-rhel8@sha256:0e273607b7d8ee6e2e542e02a2f6cfb04c144d4b70cf1fbc58d1041e26d283ab
registry.redhat.io/openshift-serverless-1/kn-serving-controller-rhel8@sha256:fdf01c170795da9598007bddf34c74e4a2b6d4c10ac2a0ad7010f30c8eb84149
registry.redhat.io/openshift-serverless-1/kn-serving-queue-rhel8@sha256:be27abd8e30d0e9b0245d5d99800290231aa246931bdbf65a757eac49f7d9ad9
registry.redhat.io/openshift-serverless-1/kn-serving-storage-version-migration-rhel8@sha256:dafcf4ee3a5836f2744e786fafd2911264a6f043d7cf17bf8cdf7b75ab9b3ff6
registry.redhat.io/openshift-serverless-1/kn-serving-webhook-rhel8@sha256:6dfc77b18f5f03fbc918f33ab5916344b546085e3cd57632d71ddb73022b5222
registry.redhat.io/openshift-serverless-1/net-istio-controller-rhel8@sha256:06100687f4d3b193fe289b45046d11bf5439f296f0c9b1e62fe16ed8624ae251
registry.redhat.io/openshift-serverless-1/net-istio-webhook-rhel8@sha256:6939d0ec31480dbfa172783d2531f6497c38dd18b0cbcc1597413e7dd49a4d62
registry.redhat.io/openshift-serverless-1/net-kourier-kourier-rhel8@sha256:1b3f3be13ff69f520ace648989ae7053b26a872af3c2baade05adfc8513f2afd
registry.redhat.io/openshift-serverless-1/serverless-ingress-rhel8@sha256:db94f6b64ac3e618c0dad70032ad3e723122d2dd566dd4099cd5f81e3f28ae8e
registry.redhat.io/openshift-serverless-1/serverless-kn-operator-rhel8@sha256:dd788378be08cd5de076fe6fe7255ec21486697197f9390c0f8afc6be0901150
registry.redhat.io/openshift-serverless-1/serverless-must-gather-rhel8@sha256:5b7aba60fba1db136c893ecdd34aa592f6079564457b6bff183218ea29f1aae1
registry.redhat.io/openshift-serverless-1/serverless-openshift-kn-rhel8-operator@sha256:9d89f51d04418acaeb36c3c0c9d6917ea29ca1d5b39df05a80da19318ea2c51c
registry.redhat.io/openshift-service-mesh/proxyv2-rhel8@sha256:8ee57a44b1fc799fd8565eb339955773bd9beedcbf46f68628ee0bd4abf26515
registry.redhat.io/openshift4/ose-kube-rbac-proxy-rhel9@sha256:92a83b201580d29aec7ee85ccc2984576c4a364b849e504225888d6f1fb9b0d2
registry.redhat.io/rhel8/buildah@sha256:3d505d9c0f5d4cd5a4ec03b8d038656c6cdbdf5191e00ce6388f7e0e4d2f1b74
registry.redhat.io/openshift-serverless-1/serverless-operator-bundle@sha256:2d675f8bf31b0cfb64503ee72e082183b7b11979d65eb636fc83f4f3a25fa5d0
```

### OpenShift Serverless Logic Operator:
```
gcr.io/kaniko-project/warmer:v1.9.0
gcr.io/kaniko-project/executor:v1.9.0
registry.redhat.io/openshift-serverless-1/logic-jobs-service-postgresql-rhel8:1.36.0
registry.redhat.io/openshift-serverless-1/logic-jobs-service-ephemeral-rhel8:1.36.0
registry.redhat.io/openshift-serverless-1/logic-data-index-postgresql-rhel8:1.36.0
registry.redhat.io/openshift-serverless-1/logic-data-index-ephemeral-rhel8:1.36.0
registry.redhat.io/openshift-serverless-1/logic-db-migrator-tool-rhel8:1.36.0
registry.redhat.io/openshift-serverless-1/logic-swf-builder-rhel8:1.36.0
registry.redhat.io/openshift-serverless-1/logic-swf-devmode-rhel8:1.36.0
registry.redhat.io/openshift-serverless-1/logic-rhel8-operator@sha256:8d3682448ebdac3aeabb2d23842b7e67a252b95f959c408af805037f9728fd3c
registry.redhat.io/openshift4/ose-kube-rbac-proxy@sha256:4564ca3dc5bac80d6faddaf94c817fbbc270698a9399d8a21ee1005d85ceda56
registry.redhat.io/openshift-serverless-1/logic-rhel8-operator@sha256:8d3682448ebdac3aeabb2d23842b7e67a252b95f959c408af805037f9728fd3c
registry.redhat.io/openshift4/ose-kube-rbac-proxy@sha256:4564ca3dc5bac80d6faddaf94c817fbbc270698a9399d8a21ee1005d85ceda56
registry.redhat.io/openshift-serverless-1/logic-operator-bundle@sha256:5fff2717f7b08df2c90a2be7bfb36c27e13be188d23546497ed9ce266f1c03f4
```

### Orchestrator Operator:
```
registry.redhat.io/rhdh-orchestrator-dev-preview-beta/controller-rhel9-operator@sha256:32e556fe067074d1f0ef0eb1f5483f62cc63d31a04c5fb2dcaea657a6471c081
registry.redhat.io/rhdh-orchestrator-dev-preview-beta/orchestrator-operator-bundle@sha256:266366306f3977ae74e1ce3d06856a709d888163bf7423b6b941adfeb8ded6c2
```

> **Note:**  
> If you encounter issues pulling images due to an invalid GPG signature, consider updating the `/etc/containers/policy.json` file to reference the appropriate beta GPG key.  
> For example, you can use:  
> `/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-beta`  
> This may be required when working with pre-release or beta images signed with a different key than the default.

## NPM packages for a disconnected environment
The packages required for the Orchestrator can be downloaded as tgz files from:
* https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator/-/backstage-plugin-orchestrator-1.6.0.tgz
* https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator-backend-dynamic/-/backstage-plugin-orchestrator-backend-dynamic-1.6.0.tgz
* https://npm.registry.redhat.com/@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic/-/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic-1.6.0.tgz
* https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator-form-widgets/-/backstage-plugin-orchestrator-form-widgets-1.6.0.tgz
  
Or using NPM packages from https://npm.registry.redhat.com e.g. by:
```bash
  npm pack "@redhat/backstage-plugin-orchestrator@1.6.0" --registry=https://npm.registry.redhat.com
  npm pack "@redhat/backstage-plugin-orchestrator-backend-dynamic@1.6.0" --registry=https://npm.registry.redhat.com
  npm pack "@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic@1.6.0" --registry=https://npm.registry.redhat.com
  npm pack "@redhat/backstage-plugin-orchestrator-form-widgets@1.6.0" --registry=https://npm.registry.redhat.com
```

# For maintainers
The images in this page were listed using the following set of commands, based on each of the operator bundle images:

## RHDH

The RHDH bundle version should match the one being used by the Orchestrator operator as pointed by the (rhdhSubscriptionStartingCSV attribute)[https://github.com/rhdhorchestrator/orchestrator-go-operator/blob/main/internal/controller/rhdh/backstage.go#L31].

The list of images was obtained by:
```bash
bash <<'EOF'
set -euo pipefail

IMG="registry.redhat.io/rhdh/rhdh-operator-bundle:1.6.1"
DIR="local-manifests-rhdh"
CSV="$DIR/rhdh-operator.clusterserviceversion.yaml"

podman pull "$IMG" --quiet >/dev/null 2>&1
BUNDLE_DIGEST=$(podman image inspect "$IMG" --format '{{ index .RepoDigests 0 }}')

podman create --name temp "$IMG" > /dev/null
podman cp temp:/manifests "$DIR"
podman rm temp > /dev/null

yq e '
  .spec.install.spec.deployments[].spec.template.spec.containers[].image,
  .spec.install.spec.deployments[].spec.template.spec.containers[].env[]
  | select(.name | test("^RELATED_IMAGE_")).value
' "$CSV" | cat - <(echo "$BUNDLE_DIGEST") | sort -u
EOF
```

## OpenShift Serverless

The list of images was obtained by:
```bash
IMG=registry.redhat.io/openshift-serverless-1/serverless-operator-bundle:1.36.0
podman run --rm --entrypoint bash "$IMG" -c "cat /manifests/serverless-operator.clusterserviceversion.yaml" | yq '.spec.relatedImages[].image' | sort | uniq
podman pull "$IMG"
podman image inspect "$IMG" --format '{{ index .RepoDigests 0 }}'
```

## OpenShift Serverless Logic

```bash
podman create --name temp-container registry.redhat.io/openshift-serverless-1/logic-operator-bundle:1.36.0-8
podman cp temp-container:/manifests ./local-manifests-osl
podman rm temp-container
yq -r '.data."controllers_cfg.yaml" | from_yaml | .. | select(tag == "!!str") | select(test("^.*\\/.*:.*$"))' ./local-manifests-osl/logic-operator-rhel8-controllers-config_v1_configmap.yaml
yq -r '.. | select(has("image")) | .image' ./local-manifests-osl/logic-operator-rhel8.clusterserviceversion.yaml
```

## Orchestrator
The list of images was obtained by:
```bash
bash <<'EOF'
set -euo pipefail

IMG="registry.redhat.io/rhdh-orchestrator-dev-preview-beta/orchestrator-operator-bundle:1.6-1751040440"
DIR="local-manifests-orchestrator"
CSV="$DIR/orchestrator-operator.clusterserviceversion.yaml"

podman pull "$IMG" --quiet >/dev/null 2>&1
BUNDLE_DIGEST=$(podman image inspect "$IMG" --format '{{ index .RepoDigests 0 }}')

podman create --name temp "$IMG" > /dev/null
podman cp temp:/manifests "$DIR"
podman rm temp > /dev/null

yq e '.spec.install.spec.deployments[].spec.template.spec.containers[].image' "$CSV" | cat - <(echo "$BUNDLE_DIGEST") | sort -u
EOF
```
