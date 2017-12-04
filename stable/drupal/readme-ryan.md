# Create infrastructure.

Made cluster with CloudSQL access, mine looked like this.

```
POST https://container.googleapis.com/v1/projects/training-dev-global/zones/europe-west1-b/clusters
{
  "cluster": {
    "name": "drupal",
    "zone": "europe-west1-b",
    "network": "default",
    "nodePools": [
      {
        "name": "default-pool",
        "initialNodeCount": 3,
        "config": {
          "machineType": "n1-standard-1",
          "imageType": "COS",
          "diskSizeGb": 100,
          "preemptible": false,
          "oauthScopes": [
            "https://www.googleapis.com/auth/compute",
            "https://www.googleapis.com/auth/devstorage.read_only",
            "https://www.googleapis.com/auth/logging.write",
            "https://www.googleapis.com/auth/monitoring",
            "https://www.googleapis.com/auth/servicecontrol",
            "https://www.googleapis.com/auth/service.management.readonly",
            "https://www.googleapis.com/auth/sqlservice.admin",
            "https://www.googleapis.com/auth/trace.append"
          ]
        },
        "autoscaling": {
          "enabled": false
        },
        "management": {
          "autoUpgrade": false,
          "autoRepair": false,
          "upgradeOptions": {}
        }
      }
    ],
    "loggingService": "none",
    "monitoringService": "none",
    "zoneLocation": "europe-west1-b",
    "initialClusterVersion": "1.7.8-gke.0",
    "masterAuth": {
      "username": "admin",
      "clientCertificateConfig": {
        "issueClientCertificate": true
      }
    },
    "subnetwork": "default",
    "legacyAbac": {
      "enabled": true
    },
    "masterAuthorizedNetworksConfig": {
      "enabled": false,
      "cidrBlocks": []
    },
    "addonsConfig": {
      "kubernetesDashboard": {
        "disabled": false
      },
      "httpLoadBalancing": {
        "disabled": false
      },
      "networkPolicyConfig": {
        "disabled": true
      }
    },
    "networkPolicy": {
      "enabled": false,
      "provider": "CALICO"
    },
    "ipAllocationPolicy": {
      "useIpAliases": false
    }
  }
}
```

Create a database, I called mine `drupal` and put it in europe-west1-b
`Done in UI`

Copy instance connection name and save it for later.
In this case it is `training-dev-global:europe-west1:drupal`

# Create service account.

Go to the service accounts page.
`https://console.cloud.google.com/iam-admin/serviceaccounts`

Select the project, sometimes GCP set no project if you follow links.

Click Create service account, call it whatever.

For Role, select Cloud SQL > Cloud SQL Client.

Click Furnish a new private key.

The default key type is JSON, which is the right one.

Click Create.

The private key file will download.

Make sure your gcloud it set to the right project `gcloud config set project [PROJECT_ID]`

Create the proxy user

`gcloud sql users create proxyuser cloudsqlproxy~% --instance=[INSTANCE_NAME] --password=[PASSWORD]`

# Create secrets

For the Service Account use the key generated earlier, mine is:
```
kubectl create secret generic cloudsql-instance-credentials \
    --from-file=credentials.json=/Users/simonince/Downloads/training-dev-global-de336ec8d01a.json --namespace=[NAMESPACE]
```

Secret needed for database access, set your password as the same as earlier.

```
kubectl create secret generic cloudsql-db-credentials \
     --from-literal=username=proxyuser --from-literal=password=[PASSWORD] --namespace=[NAMESPACE]
```

# Set vales

Set instace connection name in stable/drupal/ryan-values.yaml line 39 with the one you copied earlier.

```
##
## CloudSQL chart configuration
##

cloudsql:
  connectioName: "training-dev-global:europe-west1:drupal"
```

# Install chart

cd into `charts/stable`

package chart
`helm package drupal`

Connect to cluster
`gcloud container clusters get-credentials drupal --zone europe-west1-b --project training-dev-global`

Start tiller on cluster
`helm init`

Install chart using custom values
`helm upgrade --install --namespace [NAMESPACE] --values drupal/ryan-values.yaml [CHART_NAME] drupal-0.10.4.tgz`
