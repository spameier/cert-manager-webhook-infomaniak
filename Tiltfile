# buildifier: disable=module-docstring
load("ext://podman", "podman_build")
load('ext://helm_resource', 'helm_resource', 'helm_repo')
load('ext://namespace', 'namespace_create')

if "Podman" in str(local("docker version", quiet = True)):
    docker_build = podman_build

# setup env
debug_enabled = os.getenv("DEBUG", "false").lower()
debug_port = os.getenv("DEBUG_PORT", "30000")

infomaniak_token = os.getenv("INFOMANIAK_TOKEN")
if infomaniak_token == None:
    fail("INFOMANIAK_TOKEN env variable should be set")

acme_email = os.getenv("EMAIL")
if acme_email == None:
    fail("EMAIL env variable should be set")

test_zone = os.getenv("TEST_ZONE")
if test_zone == None:
    fail("TEST_ZONE env variable should be set")

# setup cert-manager
update_settings(k8s_upsert_timeout_secs=300)
helm_repo("jetstack", "https://charts.jetstack.io", resource_name="jetstack-repo")
helm_resource(
    "cert-manager", "jetstack/cert-manager",
    namespace="cert-manager",
    flags = [
        "--create-namespace",
        "--set",
        "crds.enabled=true"
    ]
)

# build webhook
tiltbuild_path = ".tiltbuild"
webhook_bin = tiltbuild_path + "/webhook" 
local_resource(
    "dockerfile",
    cmd = "mkdir -p {tiltbuild_path};cp Tilt.Dockerfile {tiltbuild_path}/Dockerfile".format(
        tiltbuild_path = shlex.quote(tiltbuild_path),
    ),
    deps = ["Tilt.Dockerfile"]
)
local_resource(
    "webhook_binary",
    cmd = 'CGO_ENABLED=0 go build -o {webhook_bin} -gcflags "all=-N -l" .'.format(
        webhook_bin = webhook_bin,
    ),
    deps = [
        "main.go",
        "main_test.go",
        "infomaniak_api.go",
        "go.mod",
        "go.sum",
    ],
    resource_deps = ["dockerfile"]
)
docker_build(
    "cert-manager-webhook-infomaniak",
    tiltbuild_path,
    deps = [webhook_bin],
    live_update = [
        sync(webhook_bin, "/usr/local/bin/webhook"),
        run("sh /restart.sh"),
    ],
)


# deploy webhook
namespace_create("cert-manager-infomaniak")
webhook_yaml = helm(
                    "./deploy/infomaniak-webhook",
                    name="infomaniak-webhook",
                    namespace="cert-manager-infomaniak",
                    values = ["./deploy/infomaniak-webhook/values.yaml"],
                    set = [
                        "image.repository=cert-manager-webhook-infomaniak",
                        "debug.enabled=" + debug_enabled
                    ]
)
k8s_yaml(webhook_yaml)

if debug_enabled == "true":
    k8s_resource(
        workload = "infomaniak-webhook",
        port_forwards = [port_forward(int(debug_port), int(debug_port))],
        resource_deps = ["webhook_binary"]
    )
else:
    k8s_resource(
        workload = "infomaniak-webhook",
        resource_deps = ["webhook_binary"]
    )

# deploy additional test resources
api_key_yaml = """
apiVersion: v1
kind: Secret
metadata:
  name: infomaniak-api-credentials
  namespace: cert-manager
type: Opaque
stringData:
  api-token: """ + infomaniak_token + """
"""
k8s_yaml(blob(api_key_yaml))

issuer_staging_yaml = """
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: """ + acme_email + """
    privateKeySecretRef:
      name: le-staging-account-key
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    solvers:
    - selector: {}
      dns01:
        webhook:
          groupName: acme.infomaniak.com
          solverName: infomaniak
          config:
            apiTokenSecretRef:
              name: infomaniak-api-credentials
              key: api-token
"""
k8s_yaml(blob(issuer_staging_yaml))

cert_yaml = """
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-certificate
spec:
  secretName: test-certificate-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
  - """ + test_zone + """
"""
k8s_yaml(blob(cert_yaml))
