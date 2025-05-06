---
title: "Disconnected Environment"
date: 2025-04-20
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
registry.redhat.io/rhdh/rhdh-hub-rhel9@sha256:9fd11a4551da42349809bbf34eb54c3b0ca8a3884d556593656af79e72786c01
registry.redhat.io/rhdh/rhdh-operator-bundle@sha256:c870eb3d17807a9d04011df5244ea39db66af76aefd0af68244c95ed8322d8b5
registry.redhat.io/rhdh/rhdh-rhel9-operator@sha256:df9204cfad16b43ff00385609ef4e99a292c033cb56be6ac76108cd0e0cfcb4b
registry.redhat.io/rhel9/postgresql-15@sha256:450a3c82d66f0642eee81fc3b19f8cf01fbc18b8e9dbbd2268ca1f471898db2f
```

The list of images was obtained by:
```bash
bash <<'EOF'
set -euo pipefail

IMG="registry.redhat.io/rhdh/rhdh-operator-bundle:1.5.1"
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

### OpenShift Serverless Operator:
```
registry.access.redhat.com/ubi8/nodejs-20-minimal@sha256:a2a7e399aaf09a48c28f40820da16709b62aee6f2bc703116b9345fab5830861
registry.access.redhat.com/ubi8/openjdk-21@sha256:441897a1f691c7d4b3a67bb3e0fea83e18352214264cb383fd057bbbd5ed863c
registry.access.redhat.com/ubi8/python-39@sha256:27e795fd6b1b77de70d1dc73a65e4c790650748a9cfda138fdbd194b3d6eea3d
registry.redhat.io/openshift-serverless-1/kn-backstage-plugins-eventmesh-rhel8@sha256:77665d8683230256122e60c3ec0496e80543675f39944c70415266ee5cffd080
registry.redhat.io/openshift-serverless-1/kn-client-cli-artifacts-rhel8@sha256:f983be49897be59dba1275f36bdd83f648663ee904e4f242599e9269fc354fd7
registry.redhat.io/openshift-serverless-1/kn-client-kn-rhel8@sha256:d21cc7e094aa46ba7f6ea717a3d7927da489024a46a6c1224c0b3c5834dcb7a6
registry.redhat.io/openshift-serverless-1/kn-ekb-dispatcher-rhel8@sha256:9cab1c37aae66e949a5d65614258394f566f2066dd20b5de5a8ebc3a4dd17e4c
registry.redhat.io/openshift-serverless-1/kn-ekb-kafka-controller-rhel8@sha256:e7dbf060ee40b252f884283d80fe63655ded5229e821f7af9e940582e969fc01
registry.redhat.io/openshift-serverless-1/kn-ekb-post-install-rhel8@sha256:097e7891a85779880b3e64edb2cb1579f17bc902a17d2aa0c1ef91aeb088f5f1
registry.redhat.io/openshift-serverless-1/kn-ekb-receiver-rhel8@sha256:207a1c3d7bf18a56ab8fd69255beeac6581a97576665e8b79f93df74da911285
registry.redhat.io/openshift-serverless-1/kn-ekb-webhook-kafka-rhel8@sha256:cafb9dcc4059b3bc740180cd8fb171bdad44b4d72365708d31f86327a29b9ec5
registry.redhat.io/openshift-serverless-1/kn-eventing-apiserver-receive-adapter-rhel8@sha256:ec3c038d2baf7ff915a2c5ee90c41fb065a9310ccee473f0a39d55de632293e3
registry.redhat.io/openshift-serverless-1/kn-eventing-channel-controller-rhel8@sha256:2c2912c0ba2499b0ba193fcc33360145696f6cfe9bf576afc1eac1180f50b08d
registry.redhat.io/openshift-serverless-1/kn-eventing-channel-dispatcher-rhel8@sha256:4d7ecfae62161eff86b02d1285ca9896983727ec318b0d29f0b749c4eba31226
registry.redhat.io/openshift-serverless-1/kn-eventing-controller-rhel8@sha256:1b4856760983e14f50028ab3d361bb6cd0120f0be6c76b586f2b42f5507c3f63
registry.redhat.io/openshift-serverless-1/kn-eventing-filter-rhel8@sha256:cec64e69a3a1c10bc2b48b06a5dd6a0ddd8b993840bbf1ac7881d79fc854bc91
registry.redhat.io/openshift-serverless-1/kn-eventing-ingress-rhel8@sha256:7e6049da45969fa3f766d2a542960b170097b2087cad15f5bba7345d8cdc0dad
registry.redhat.io/openshift-serverless-1/kn-eventing-istio-controller-rhel8@sha256:d14fd8abf4e8640dbde210f567dd36866fe5f0f814a768a181edcb56a8e7f35b
registry.redhat.io/openshift-serverless-1/kn-eventing-jobsink-rhel8@sha256:8ecea4b6af28fe8c7f8bfcc433c007555deb8b7def7c326867b04833c524565d
registry.redhat.io/openshift-serverless-1/kn-eventing-migrate-rhel8@sha256:e408db39c541a46ebf7ff1162fe6f81f6df1fe4eeed4461165d4cb1979c63d27
registry.redhat.io/openshift-serverless-1/kn-eventing-mtchannel-broker-rhel8@sha256:2685917be6a6843c0d82bddf19f9368c39c107dae1fd1d4cb2e69d1aa87588ec
registry.redhat.io/openshift-serverless-1/kn-eventing-mtping-rhel8@sha256:c5a5b6bc4fdb861133fd106f324cc4a904c6c6a32cabc6203efc578d8f46bbf4
registry.redhat.io/openshift-serverless-1/kn-eventing-webhook-rhel8@sha256:efe2d60e777918df9271f5512e4722f8cf667fe1a59ee937e093224f66bc8cbf
registry.redhat.io/openshift-serverless-1/kn-plugin-event-sender-rhel8@sha256:08f0b4151edd6d777e2944c6364612a5599e5a775e5150a76676a45f753c2e23
registry.redhat.io/openshift-serverless-1/kn-plugin-func-func-util-rhel8@sha256:01e0ab5c8203ef0ca39b4e9df8fd1a8c2769ef84fce7fecefc8e8858315e71ca
registry.redhat.io/openshift-serverless-1/kn-serving-activator-rhel8@sha256:3892eadbaa6aba6d79d6fe2a88662c851650f7c7be81797b2fc91d0593a763d1
registry.redhat.io/openshift-serverless-1/kn-serving-autoscaler-hpa-rhel8@sha256:6b30d3f6d77a6e74d4df5a9d2c1b057cdc7ebbbf810213bc0a97590e741bae1c
registry.redhat.io/openshift-serverless-1/kn-serving-autoscaler-rhel8@sha256:00777fa53883f25061ebe171b0d47025d27acd39582a619565e9167288321952
registry.redhat.io/openshift-serverless-1/kn-serving-controller-rhel8@sha256:41a21fdc683183422ebb29707d81eca96d7ca119d01f369b9defbaea94c09939
registry.redhat.io/openshift-serverless-1/kn-serving-queue-rhel8@sha256:bd464d68e283ce6c48ae904010991b491b738ada5a419f044bf71fd48326005b
registry.redhat.io/openshift-serverless-1/kn-serving-storage-version-migration-rhel8@sha256:de87597265ee5ac26db4458a251d00a5ec1b5cd0bfff4854284070fdadddb7ab
registry.redhat.io/openshift-serverless-1/kn-serving-webhook-rhel8@sha256:eb33e874b5a7c051db91cd6a63223aabd987988558ad34b34477bee592ceb3ab
registry.redhat.io/openshift-serverless-1/net-istio-controller-rhel8@sha256:ec77d44271ba3d86af6cbbeb70f20a720d30d1b75e93ac5e1024790448edf1dd
registry.redhat.io/openshift-serverless-1/net-istio-webhook-rhel8@sha256:07074f52b5fb1f2eb302854dce1ed5b81c665ed843f9453fc35a5ebcb1a36696
registry.redhat.io/openshift-serverless-1/net-kourier-kourier-rhel8@sha256:e5f1111791ffff7978fe175f3e3af61a431c08d8eea4457363c66d66596364d8
registry.redhat.io/openshift-serverless-1/serverless-ingress-rhel8@sha256:3d1ab23c9ce119144536dd9a9b80c12bf2bb8e5f308d9c9c6c5b48c41f4aa89e
registry.redhat.io/openshift-serverless-1/serverless-kn-operator-rhel8@sha256:78cb34062730b3926a465f0665475f0172a683d7204423ec89d32289f5ee329d
registry.redhat.io/openshift-serverless-1/serverless-must-gather-rhel8@sha256:119fbc185f167f3866dbb5b135efc4ee787728c2e47dd1d2d66b76dc5c43609e
registry.redhat.io/openshift-serverless-1/serverless-openshift-kn-rhel8-operator@sha256:0f763b740cc1b614cf354c40f3dc17050e849b4cbf3a35cdb0537c2897d44c95
registry.redhat.io/openshift-service-mesh/proxyv2-rhel8@sha256:b30d60cd458133430d4c92bf84911e03cecd02f60e88a58d1c6c003543cf833a
registry.redhat.io/openshift4/ose-kube-rbac-proxy-rhel9@sha256:3fcd8e2bf0bcb8ff8c93a87af2c59a3bcae7be8792f9d3236c9b5bbd9b6db3b2
registry.redhat.io/rhel8/buildah@sha256:3d505d9c0f5d4cd5a4ec03b8d038656c6cdbdf5191e00ce6388f7e0e4d2f1b74
registry.redhat.io/source-to-image/source-to-image-rhel8@sha256:6a6025914296a62fdf2092c3a40011bd9b966a6806b094d51eec5e1bd5026ef4
registry.redhat.io/openshift-serverless-1/serverless-operator-bundle@sha256:93b945eb2361b07bc86d67a9a7d77a0301a0bad876c83a9a64af2cfb86c83bff
```

The list of images was obtained by:
```bash
IMG=registry.redhat.io/openshift-serverless-1/serverless-operator-bundle:1.35.0
podman run --rm --entrypoint bash "$IMG" -c "cat /manifests/serverless-operator.clusterserviceversion.yaml" | yq '.spec.relatedImages[].image' | sort | uniq
podman pull "$IMG"
podman image inspect "$IMG" --format '{{ index .RepoDigests 0 }}'
```

### OpenShift Serverless Logic Operator:
```
registry.redhat.io/openshift-serverless-1/logic-operator-bundle@sha256:a1d1995b2b178a1242d41f1e8df4382d14317623ac05b91bf6be971f0ac5a227
registry.redhat.io/openshift-serverless-1/logic-jobs-service-postgresql-rhel8:1.35.0
registry.redhat.io/openshift-serverless-1/logic-jobs-service-ephemeral-rhel8:1.35.0
registry.redhat.io/openshift-serverless-1/logic-data-index-postgresql-rhel8:1.35.0
registry.redhat.io/openshift-serverless-1/logic-data-index-ephemeral-rhel8:1.35.0
registry.redhat.io/openshift-serverless-1/logic-swf-builder-rhel8:1.35.0
registry.redhat.io/openshift-serverless-1/logic-swf-devmode-rhel8:1.35.0
registry.redhat.io/openshift4/ose-kube-rbac-proxy@sha256:4564ca3dc5bac80d6faddaf94c817fbbc270698a9399d8a21ee1005d85ceda56
registry.redhat.io/openshift-serverless-1/logic-rhel8-operator@sha256:203043ca27819f7d039fd361d0816d5a16d6b860ff19d737b07968ddfba3d2cd
registry.redhat.io/openshift4/ose-kube-rbac-proxy@sha256:4564ca3dc5bac80d6faddaf94c817fbbc270698a9399d8a21ee1005d85ceda56
registry.redhat.io/openshift4/ose-cli:latest

gcr.io/kaniko-project/warmer:v1.9.0
gcr.io/kaniko-project/executor:v1.9.0
```

```bash
podman create --name temp-container registry.redhat.io/openshift-serverless-1/logic-operator-bundle:1.35.0-5
podman cp temp-container:/manifests ./local-manifests-osl
podman rm temp-container
yq -r '.data."controllers_cfg.yaml" | from_yaml | .. | select(tag == "!!str") | select(test("^.*\\/.*:.*$"))' ./local-manifests-osl/logic-operator-rhel8-controllers-config_v1_configmap.yaml
yq -r '.. | select(has("image")) | .image' ./local-manifests-osl/logic-operator-rhel8.clusterserviceversion.yaml
```

### Orchestrator Operator:
```
registry.redhat.io/rhdh-orchestrator-dev-preview-beta/controller-rhel9-operator@sha256:ea42a1a593af9433ac74e58269c7e0705a08dbfa8bd78fba69429283a307131a
registry.redhat.io/rhdh-orchestrator-dev-preview-beta/orchestrator-operator-bundle@sha256:0a9e5d2626b4306c57659dbb90e160f1c01d96054dcac37f0975500d2c22d9c7
```

The list of images was obtained by:
```bash
bash <<'EOF'
set -euo pipefail

IMG="registry.redhat.io/rhdh-orchestrator-dev-preview-beta/orchestrator-operator-bundle:1.5-1744669755"
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

> **Note:**  
> If you encounter issues pulling images due to an invalid GPG signature, consider updating the `/etc/containers/policy.json` file to reference the appropriate beta GPG key.  
> For example, you can use:  
> `/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-beta`  
> This may be required when working with pre-release or beta images signed with a different key than the default.

## NPM packages for a disconnected environment
The packages required for the Orchestrator can be downloaded as tgz files from:
* https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator/-/backstage-plugin-orchestrator-1.5.1.tgz
* https://npm.registry.redhat.com/@redhat/backstage-plugin-orchestrator-backend-dynamic/-/backstage-plugin-orchestrator-backend-dynamic-1.5.1.tgz
* https://npm.registry.redhat.com/@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic/-/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic-1.5.1.tgz
  
Or using NPM packages from https://npm.registry.redhat.com e.g. by:
```bash
  npm pack "@redhat/backstage-plugin-orchestrator@1.5.1" --registry=https://npm.registry.redhat.com
  npm pack "@redhat/backstage-plugin-orchestrator-backend-dynamic@1.5.1" --registry=https://npm.registry.redhat.com
  npm pack "@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic@1.5.1" --registry=https://npm.registry.redhat.com
```
