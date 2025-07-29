# rancher-api
Manage cluster K3s &amp; RKE2 with API

## Membuat cluster K3s
```json
{
  "type": "provisioning.cattle.io.cluster",
  "metadata": {
    "name": "febri4n-k3s-api",
    "namespace": "fleet-default",
    "labels": {
      "provider.cattle.io": "k3s"
    }
  },
  "spec": {
    "kubernetesVersion": "v1.30.13+k3s1",
    "enableNetworkPolicy": false,
    "rkeConfig": {
      "machineGlobalConfig": {
        "disable": ["servicelb", "traefik"],
        "disable-kube-proxy": false,
        "secrets-encryption": false
      },
      "etcd": {
        "snapshotRetention": 14,
        "snapshotScheduleCron": "0 */5 * * *"
      },
      "upgradeStrategy": {
        "controlPlaneConcurrency": "1",
        "workerConcurrency": "1",
        "controlPlaneDrainOptions": {
          "enabled": false,
          "force": false
        },
        "workerDrainOptions": {
          "enabled": false,
          "force": false
        }
      },
      "machineSelectorConfig": [
        {
          "config": {
            "docker": false,
            "protect-kernel-defaults": false
          }
        }
      ]
    }
  }
}
```
```bash
curl -X POST \
  'https://example.com/v1/provisioning.cattle.io.clusters/fleet-default' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  --data-binary @new-cluster-k3s.json
```
## Membuat cluster RKE2
```json
{
  "type": "provisioning.cattle.io.cluster",
  "metadata": {
    "name": "febri4n-rke2-api",
    "namespace": "fleet-default",
    "labels": {
      "provider.cattle.io": "rke2"
    }
  },
  "spec": {
    "kubernetesVersion": "v1.31.9+rke2r1",
    "rkeConfig": {
      "machineGlobalConfig": {
        "cni": "canal",
        "disable": ["rke2-ingress-nginx"],
        "disable-kube-proxy": false,
        "etcd-expose-metrics": false
      },
      "etcd": {
        "snapshotRetention": 5,
        "snapshotScheduleCron": "0 */5 * * *"
      },
      "upgradeStrategy": {
        "controlPlaneConcurrency": "1",
        "workerConcurrency": "1",
        "controlPlaneDrainOptions": {
          "enabled": false,
          "force": false,
          "gracePeriod": -1
        },
        "workerDrainOptions": {
          "enabled": false,
          "force": false,
          "gracePeriod": -1
        }
      }
    }
  }
}
```
```bash
curl -X POST \
  'https://example.com/v1/provisioning.cattle.io.clusters/fleet-default' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  --data-binary @new-cluster-rke2.json
```
## Mendapatkan registration command cluster
```bash
# Ganti <Cluster_ID> & <Token_Rancher>
# Versi 1, dia akan mendapatkan lebih dari 1 node command tergantung jumlah token active
curl -X GET \
  'https://example.com/v3/clusterregistrationtokens?clusterId=<Cluster_ID>' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  -H 'Accept: application/json' \
  | jq -r '.data[] | .nodeCommand'
```
```bash
# Ganti <Cluster_ID> & <Token_Rancher>
# Versi 2, dia akan mengambil token terakhir yang masih active (jika lebih dari 1 active)
curl -X GET \
  'https://example.com/v3/clusterregistrationtokens?clusterId=<Cluster_ID>' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  -H 'Accept: application/json' \
  | jq -r '.data | sort_by(.created) | last | .nodeCommand'
```
## Mendapatkan list clusterId dan clusterName yang ada di rancher
```bash
curl -s -X GET \
  'https://example.com/v3/clusters' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  -H 'Accept: application/json' \
  | jq -r '
    ["CLUSTER ID", "CLUSTER NAME"],
    (.data[] | [.id, .name]) |
    @tsv' \
  | column -t -s $'\t'
```
## Menghapus cluster yang ada di rancher
```bash
curl -X DELETE \
  'https://example.com/v3/clusters/<Cluster_ID>' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  -H 'Accept: application/json'
```
## Detail cluster
```bash
curl -X GET \
  'https://example.com/v3/clusters/<Cluster_ID>' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  -H 'Accept: application/json'
```
## List Machine Cluster
```bash
# Versi 1, berdasarkan node cluster
curl -s -X GET \
  'https://example.com/v3/clusters/<Cluster_ID>/nodes' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  -H 'Accept: application/json' \
  | jq -r '
    ["ID", "HOSTNAME", "IP ADDRESS", "ETCD", "CONTROLPLANE", "WORKER"],
    (.data[] | [.id, .hostname, .ipAddress, .etcd, .controlPlane, .worker]) |
    @tsv' \
  | column -t -s $'\t'
```
```bash
# Versi 2, berdasarkan machines cluster
curl -s -X GET \
  'https://example.com/v1/cluster.x-k8s.io.machines' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  | jq -r '
    ["ID", "CLUSTER NAME", "CONTROLPLANE", "ETCD", "WORKER", "OS", "HOSTNAME", "IP ADDRESS"],
    (.data[] | select(.metadata.labels["cluster.x-k8s.io/cluster-name"] == "<Cluster_Name>") | [
      .id,
      .metadata.labels["cluster.x-k8s.io/cluster-name"],
      (.metadata.labels["rke.cattle.io/control-plane-role"] // "false"),
      (.metadata.labels["rke.cattle.io/etcd-role"] // "false"),
      (.metadata.labels["rke.cattle.io/worker-role"] // "false"),
      .status.nodeInfo.osImage,
      (.status.addresses[] | select(.type == "Hostname").address),
      (.status.addresses[] | select(.type == "InternalIP").address)
    ]) | @tsv' \
  | column -t -s $'\t'
```
## Delete Machine Cluster
```bash
curl -X DELETE \
  "https://example.com/v1/cluster.x-k8s.io.machines/<Machine_Id>" \
  -H "Authorization: Bearer <Token_Rancher>"
```
## Mendapatkan Kubeconfig
```bash
curl -X POST \
  'https://example.com/v3/clusters/<Cluster_ID>?action=generateKubeconfig' \
  -H 'Authorization: Bearer <Token_Rancher>' \
  -H 'Accept: application/json' \
  | jq -r '.config' > kubeconfig.yaml
```
### Mendapatkan list etcdSnapshot
```bash
curl -s -X GET \
  "https://rancher-staging.febri4n.my.id/v1/rke.cattle.io.etcdsnapshots" \
  -H "Authorization: Bearer <Token_Rancher>" \
  | jq -r --arg CLUSTER "<Cluster_Name>" '
    ("FULL_ID\tSNAPSHOT_NAME\tCLUSTER\tNODE\tCREATED_AT\tSIZE_MB\tSTATUS\tSTORAGE_TYPE"),
    (.data[] | select(.spec.clusterName == $CLUSTER) | 
    "\(.metadata.namespace)/\(.metadata.name)\t\(.metadata.annotations."rke.cattle.io/snapshot-name" // .metadata.name)\t\(.spec.clusterName)\t\(.snapshotFile.nodeName)\t\(.snapshotFile.createdAt)\t\(
      .snapshotFile.size / (1024*1024) | floor)\t\(.snapshotFile.status)\t\(
      .metadata.annotations."etcdsnapshot.rke.io/storage")")' \
  | column -t -s $'\t'
```
