# Ocp

``bash
############################################
# 0) Set variables (edit these)
############################################
export OCP_VERSION="4.17.0"              # or 4.17.z
export ARCH="x86_64"
export CLUSTER_NAME="sno"
export BASE_DOMAIN="example.com"
export NODE_HOSTNAME="sno-0"
export NODE_MAC="aa:bb:cc:dd:ee:ff"
export NODE_IP="192.168.1.50"            # node IP (also used for api/apps in DNS)
export GATEWAY="192.168.1.1"
export DNS_SERVER="192.168.1.10"

export MIRROR_REG="registry.local:5000"
export MIRROR_NS="ocp4"                  # namespace/repo prefix in your registry
export PULLSECRET="$HOME/pull-secret.json"

# work dirs
export WORK="$HOME/sno-work"
mkdir -p "$WORK" && cd "$WORK"


############################################
# 1) Mirror the OpenShift release to your internal registry (run on a host that can reach upstream registries)
#    (Preferred: oc-mirror workflow for disconnected environments)
############################################

# Create an ImageSetConfiguration (release-only example)
cat > imageset-config.yaml <<EOF
apiVersion: mirror.openshift.io/v1alpha2
kind: ImageSetConfiguration
storageConfig:
  registry:
    imageURL: ${MIRROR_REG}/mirror/metadata
mirror:
  platform:
    channels:
    - name: stable-${OCP_VERSION%.*}     # stable-4.17
      minVersion: ${OCP_VERSION}
      maxVersion: ${OCP_VERSION}
EOF

# Mirror to internal registry
oc mirror --config=imageset-config.yaml docker://${MIRROR_REG} --dest-skip-tls=false

# oc-mirror writes results under ./oc-mirror-workspace/ (keep it; you’ll use generated resources later)


############################################
# 2) Create install-config.yaml (SNO)
############################################
cat > install-config.yaml <<EOF
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: ${CLUSTER_NAME}
compute:
- name: worker
  replicas: 0
controlPlane:
  name: master
  replicas: 1
networking:
  networkType: OVNKubernetes
pullSecret: '$(cat "${PULLSECRET}")'
sshKey: '$(cat "$HOME/.ssh/id_rsa.pub")'
EOF


############################################
# 3) Create agent-config.yaml (single node; static IP example via NMState)
############################################
cat > agent-config.yaml <<EOF
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentConfig
metadata:
  name: ${CLUSTER_NAME}
rendezvousIP: ${NODE_IP}
hosts:
  - hostname: ${NODE_HOSTNAME}
    role: master
    interfaces:
      - name: eth0
        macAddress: ${NODE_MAC}
    networkConfig:
      interfaces:
        - name: eth0
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: ${NODE_IP}
                prefix-length: 24
            dhcp: false
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${GATEWAY}
            next-hop-interface: eth0
      dns-resolver:
        config:
          server:
            - ${DNS_SERVER}
EOF


############################################
# 4) (Disconnected) Add mirror registry trust + mirror resources into the install dir
#    - If your registry uses a private CA, put CA bundle here (example file path):
############################################
# mkdir -p manifests
# cp /path/to/registry-ca.pem ./additional-trust-bundle.pem
# (Then reference it in install-config.yaml as additionalTrustBundle if needed)


############################################
# 5) Create the agent ISO
############################################
openshift-install agent create image --dir "$WORK"


############################################
# 6) Boot the physical machine from the generated ISO (out-of-band / USB)
#    Then, from this same workstation, wait for completion:
############################################
openshift-install agent wait-for bootstrap-complete --dir "$WORK"
openshift-install agent wait-for install-complete --dir "$WORK"


############################################
# 7) Use the kubeconfig after install
############################################
export KUBECONFIG="$WORK/auth/kubeconfig"
oc get nodes
oc get clusterversion


``