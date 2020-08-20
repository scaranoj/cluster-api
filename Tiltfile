# -*- mode: Python -*-

# set defaults

envsubst_cmd = "./hack/tools/bin/envsubst"

settings = {
    "deploy_cert_manager": True,
    "preload_images_for_kind": True,
    "enable_providers": ["docker"],
    "kind_cluster_name": "kind",
}

# global settings
settings.update(read_json(
    "tilt-settings.json",
    default = {},
))

if settings.get("trigger_mode") == "manual":
    trigger_mode(TRIGGER_MODE_MANUAL)

allow_k8s_contexts(settings.get("allowed_contexts"))

default_registry(settings.get("default_registry"))

always_enable_providers = ["core"]
extra_args = settings.get("extra_args", {})

providers = {
    "core": {
        "context": ".",
        "image": "gcr.io/k8s-staging-cluster-api/cluster-api-controller",
        "live_reload_deps": [
            "main.go",
            "go.mod",
            "go.sum",
            "api",
            "cmd",
            "controllers",
            "errors",
            "third_party",
            "util",
            "exp",
            "feature",
        ],
    },
    "kubeadm-bootstrap": {
        "context": "bootstrap/kubeadm",
        "image": "gcr.io/k8s-staging-cluster-api/kubeadm-bootstrap-controller",
        "live_reload_deps": [
            "main.go",
            "api",
            "controllers",
            "internal",
        ],
    },
    "kubeadm-control-plane": {
        "context": "controlplane/kubeadm",
        "image": "gcr.io/k8s-staging-cluster-api/kubeadm-control-plane-controller",
        "live_reload_deps": [
            "main.go",
            "api",
            "controllers",
            "internal",
        ],
    },
    "docker": {
        "context": "test/infrastructure/docker",
        "image": "gcr.io/k8s-staging-cluster-api/capd-manager",
        "live_reload_deps": [
            "main.go",
            "go.mod",
            "go.sum",
            "api",
            "cloudinit",
            "controllers",
            "docker",
            "third_party",
        ],
        "additional_docker_helper_commands": """
RUN wget -qO- https://dl.k8s.io/v1.14.4/kubernetes-client-linux-amd64.tar.gz | tar xvz
RUN wget -qO- https://get.docker.com | sh
""",
        "additional_docker_build_commands": """
COPY --from=tilt-helper /usr/bin/docker /usr/bin/docker
COPY --from=tilt-helper /go/kubernetes/client/bin/kubectl /usr/bin/kubectl
""",
    },
}

# Reads a provider's tilt-provider.json file and merges it into the providers map.
# A list of dictionaries is also supported by enclosing it in brackets []
# An example file looks like this:
# {
#     "name": "aws",
#     "config": {
#         "image": "gcr.io/k8s-staging-cluster-api-aws/cluster-api-aws-controller",
#         "live_reload_deps": [
#             "main.go", "go.mod", "go.sum", "api", "cmd", "controllers", "pkg"
#         ]
#     }
# }
def load_provider_tiltfiles():
    provider_repos = settings.get("provider_repos", [])

    for repo in provider_repos:
        file = repo + "/tilt-provider.json"
        provider_details = read_json(file, default = {})
        if type(provider_details) != type([]):
            provider_details = [provider_details]
        for item in provider_details:
            provider_name = item["name"]
            provider_config = item["config"]
            if "context" in provider_config:
                provider_config["context"] = repo + "/" + provider_config["context"]
            else:
                provider_config["context"] = repo
            providers[provider_name] = provider_config

tilt_helper_dockerfile_header = """
# Tilt image
FROM golang:1.13.14 as tilt-helper
# Support live reloading with Tilt
RUN wget --output-document /restart.sh --quiet https://raw.githubusercontent.com/windmilleng/rerun-process-wrapper/master/restart.sh  && \
    wget --output-document /start.sh --quiet https://raw.githubusercontent.com/windmilleng/rerun-process-wrapper/master/start.sh && \
    chmod +x /start.sh && chmod +x /restart.sh
"""

tilt_dockerfile_header = """
FROM gcr.io/distroless/base:debug as tilt
WORKDIR /
COPY --from=tilt-helper /start.sh .
COPY --from=tilt-helper /restart.sh .
COPY manager .
"""

cert_manager_test_resources = """
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
"""

# Configures a provider by doing the following:
#
# 1. Enables a local_resource go build of the provider's manager binary
# 2. Configures a docker build for the provider, with live updating of the manager binary
# 3. Runs kustomize for the provider's config/ and applies it
def enable_provider(name):
    p = providers.get(name)

    context = p.get("context")

    # Prefix each live reload dependency with context. For example, for if the context is
    # test/infra/docker and main.go is listed as a dep, the result is test/infra/docker/main.go. This adjustment is
    # needed so Tilt can watch the correct paths for changes.
    live_reload_deps = []
    for d in p.get("live_reload_deps", []):
        live_reload_deps.append(context + "/" + d)

    # Set up a local_resource build of the provider's manager binary. The provider is expected to have a main.go in
    # manager_build_path. The binary is written to .tiltbuild/manager.
    local_resource(
        name + "_manager",
        cmd = "cd " + context + ';mkdir -p .tiltbuild;CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags \'-extldflags "-static"\' -o .tiltbuild/manager',
        deps = live_reload_deps,
    )

    additional_docker_helper_commands = p.get("additional_docker_helper_commands", "")
    additional_docker_build_commands = p.get("additional_docker_build_commands", "")

    dockerfile_contents = "\n".join([
        tilt_helper_dockerfile_header,
        additional_docker_helper_commands,
        tilt_dockerfile_header,
        additional_docker_build_commands,
    ])

    # Set up an image build for the provider. The live update configuration syncs the output from the local_resource
    # build into the container.
    entrypoint = ["sh", "/start.sh", "/manager"]
    provider_args = extra_args.get(name)
    if provider_args:
        entrypoint.extend(provider_args)

    docker_build(
        ref = p.get("image"),
        context = context + "/.tiltbuild/",
        dockerfile_contents = dockerfile_contents,
        target = "tilt",
        entrypoint = entrypoint,
        only = "manager",
        live_update = [
            sync(context + "/.tiltbuild/manager", "/manager"),
            run("sh /restart.sh"),
        ],
    )

    # Copy all the substitutions from the user's tilt-settings.json into the environment. Otherwise, the substitutions
    # are not available and their placeholders will be replaced with the empty string when we call kustomize +
    # envsubst below.
    substitutions = settings.get("kustomize_substitutions", {})
    os.environ.update(substitutions)

    # Apply the kustomized yaml for this provider
    yaml = str(kustomize_with_envsubst(context + "/config"))
    k8s_yaml(blob(yaml))

# Prepull all the cert-manager images to your local environment and then load them directly into kind. This speeds up
# setup if you're repeatedly destroying and recreating your kind cluster, as it doesn't have to pull the images over
# the network each time.
def deploy_cert_manager():
    registry = settings.get("cert_manager_registry", "quay.io/jetstack")
    version = settings.get("cert_manager_version", "v0.16.1")

    # check if cert-mamager is already installed, otherwise pre-load images & apply the manifest
    # NB. this is required until https://github.com/jetstack/cert-manager/issues/3121 is addressed otherwise
    # when applying the manifest twice to same cluster kubectl get stuck
    existsCheck = str(local("kubectl get namespaces"))
    if existsCheck.find("cert-manager") == -1:
        # pre-load cert-manager images in kind
        images = ["cert-manager-controller", "cert-manager-cainjector", "cert-manager-webhook"]
        if settings.get("preload_images_for_kind"):
            for image in images:
                local("docker pull {}/{}:{}".format(registry, image, version))
                local("kind load docker-image --name {} {}/{}:{}".format(settings.get("kind_cluster_name"), registry, image, version))

        # apply the cert-manager manifest
        local("kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/{}/cert-manager.yaml".format(version))

    # verifies cert-manager is properly working (https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation)
    # 1. wait for the cert-manager to be running
    local("kubectl wait --for=condition=Available --timeout=300s -n cert-manager deployment/cert-manager")
    local("kubectl wait --for=condition=Available --timeout=300s -n cert-manager deployment/cert-manager-cainjector")
    local("kubectl wait --for=condition=Available --timeout=300s -n cert-manager deployment/cert-manager-webhook")

    # 2. create a test certificate
    local("cat << EOF | kubectl apply -f - " + cert_manager_test_resources + "EOF")
    local("kubectl wait --for=condition=Ready --timeout=300s -n cert-manager-test certificate/selfsigned-cert ")
    local("cat << EOF | kubectl delete -f - " + cert_manager_test_resources + "EOF")

# Users may define their own Tilt customizations in tilt.d. This directory is excluded from git and these files will
# not be checked in to version control.
def include_user_tilt_files():
    user_tiltfiles = listdir("tilt.d")
    for f in user_tiltfiles:
        include(f)

# Enable core cluster-api plus everything listed in 'enable_providers' in tilt-settings.json
def enable_providers():
    local("make envsubst")
    user_enable_providers = settings.get("enable_providers", [])
    union_enable_providers = {k: "" for k in user_enable_providers + always_enable_providers}.keys()
    for name in union_enable_providers:
        enable_provider(name)

def kustomize_with_envsubst(path):
    return str(local("kustomize build {} | {}".format(path, envsubst_cmd), quiet = True))

##############################
# Actual work happens here
##############################
include_user_tilt_files()

load_provider_tiltfiles()

if settings.get("deploy_cert_manager"):
    deploy_cert_manager()

enable_providers()
