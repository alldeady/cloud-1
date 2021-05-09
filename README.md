# WordPress on Google Kubernetes Engine with cloud SQL, filestore and CDN

- Fast loading
- Heavy traffic
- Auto-scaling

## Architecture diagram
![Image alt](https://github.com/alldeady/cloud-1/blob/main/architecture-diagram.jpg)
## Deploy commands for cloud shell
```
WORKING_DIR=$(pwd)
export PROJECT_ID=<your-project-id>
```

## Create GKE
```
CLUSTER_NAME=cloud-1-cluster
gcloud beta container --project $PROJECT_ID \
    clusters create $CLUSTER_NAME --region "europe-central2" \
    --no-enable-basic-auth --cluster-version "1.18.17-gke.100" \
    --release-channel "regular" --machine-type "e2-medium" \
    --image-type "COS" --disk-type "pd-standard" --disk-size "20" \
    --metadata disable-legacy-endpoints=true \
    --scopes "https://www.googleapis.com/auth/devstorage.read_only",\
    "https://www.googleapis.com/auth/logging.write",\
    "https://www.googleapis.com/auth/monitoring",\
    "https://www.googleapis.com/auth/servicecontrol",\
    "https://www.googleapis.com/auth/service.management.readonly",\
    "https://www.googleapis.com/auth/trace.append" \
    --max-pods-per-node "8" --num-nodes "3" \
    --enable-stackdriver-kubernetes --enable-ip-alias \
    --network "projects/$PROJECT_ID/global/networks/default" \
    --subnetwork "projects/$PROJECT_ID/regions/europe-central2/subnetworks/default" \
    --no-enable-intra-node-visibility --default-max-pods-per-node "8" --enable-autoscaling \
    --min-nodes "1" --max-nodes "3" --no-enable-master-authorized-networks \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
    --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 \
    --max-unavailable-upgrade 0 --autoscaling-profile optimize-utilization \
    --enable-shielded-nodes --node-locations "europe-central2-a","europe-central2-b","europe-central2-c"
```

## Create cloud SQL
```
INSTANCE_NAME=cloud-1-sql
gcloud sql instances create $INSTANCE_NAME

INSTANCE_CONNECTION_NAME=$(gcloud sql instances describe $INSTANCE_NAME \
    --format='value(connectionName)')
gcloud sql databases create wordpress --instance $INSTANCE_NAME

CLOUD_SQL_PASSWORD=wordpress
gcloud sql users create wordpress --host=% --instance $INSTANCE_NAME \
    --password $CLOUD_SQL_PASSWORD
```

## Create creds for cloud SQL proxy
```
SA_NAME=cloudsql-proxy
gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME

SA_EMAIL=$(gcloud iam service-accounts list \
    --filter=displayName:$SA_NAME \
    --format='value(email)')

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --role roles/cloudsql.client \
    --member serviceAccount:$SA_EMAIL

gcloud iam service-accounts keys create $WORKING_DIR/key.json \
    --iam-account $SA_EMAIL

kubectl create secret generic cloudsql-db-credentials \
    --from-literal username=wordpress \
    --from-literal password=$CLOUD_SQL_PASSWORD

kubectl create secret generic cloudsql-instance-credentials \
    --from-file $WORKING_DIR/key.json
```

## Create filestore
```
gcloud filestore instances create nfs-server
    --project=cloud1-313114 \
    --zone=europe-central2-a \
    --tier=STANDARD \
    --file-share=name="vol1",capacity=1TB \
    --network=name="default",reserved-ip-range="10.0.0.0/29"

export IP_ADDRESS_FILESTORE=$(gcloud filestore instances list --format='value(IP_ADDRESS)')
cat $WORKING_DIR/pv.yaml.template | envsubst > $WORKING_DIR/pv.yaml

kubectl create -f pv.yaml

kubectl create -f pvc.yaml
```

## Deploy wordpress
```
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install wp bitnami/wordpress -f wp-values.yaml
```

## Create ingress
```
kubectl create -f ingress.yaml
```

## Enable Cloud CDN
```
BACKEND_SERVICE=$(gcloud compute backend-services list --format='value(NAME)')
gcloud compute backend-services update $BACKEND_SERVICE \
    --enable-cdn \
    --cache-mode="CACHE_All_STATIC"
```

## Done :)
