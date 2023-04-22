# Installation Notes:

## GCP

#### Set up Cloud DNS

1. Create Zone and update NameCheap with generated NS. 
This results in 2 Google Cloud DNS Zone entries (SOA and NS)

#### Create Clusters

For GCP multi-cluster setup:

1. Clone https://github.com/asaikali/tap-envs
2. `cd tap-envs/envs/gcp-ref-arch`
3. Create `settings.sh` with env vars mentioned in the script
4. Run: `./01-gcp-ref-arch-infra.sh create`
5. Run: `./01-gcp-ref-arch-infra.sh creds`
6. Copy kube-config files generated in `state` folder to `~/.kube` or `~/.secrets/<CODE-NAME>`. Optionally merge with `~/.kube/config` file.
7. Copy service-account file generated in `state` folder to `~/.secrets/<CODE-NAME>` file.

> Note: To delete the clusters, run: `./01-gcp-ref-arch-infra.sh delete`

#### Follow Docs

1. Install cluster-essentials
2. Create new cluster entries in gitops repo
3. Create tap-values.yaml file for each cluster
4. Create tap-sensitive-values.sops.yaml file for each cluster (make sure env vars are set first -- see ./tanzu-sync/scripts/sensitive-values.sh, use ~/.secretes/tap-gitops/setenv.sh))
5. Add any custom files (e.g. git-ssh secret, registry-credentials secret and secret exporter; see files in ~/.secrets/tap-gitops/tap-demo)
6. Target proper cluster using `kubectx` or `kubectl config...`
7. Set env vars (source ~/.secrets/tap-gitops/setenv.sh) and run configure script (./tanzu-sync/scripts/configure.sh) 
8. Run deploy script (./tanzu-sync/scripts/deploy.sh)
9. Update Cloud DNS with external IP (`kubectl get service -n tanzu-system-ingress`)
10. Check status (`kapp list -A`)

Set up Namespace Provisioner

# Create registry-crednetials secret
```
# Create secret "registry-credentials"
kubectl create secret docker-registry -n tap-install --dry-run=client registry-credentials \
        --docker-username='_json_key' \
        --docker-password="$(cat gcp-service-account.json )" \
        --docker-server='us-central1-docker.pkg.dev' -o yaml > registry-credentials-secret.yaml
```

# Create git-ssh secret
https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-git-auth.html?hWord=N4IghgNiBcIM5wBYgL5Agit

### TLS Setup

For each CLUSTER (set $CLUSTER to view, iterate, build, etc...):

1. Generate cert
```
sudo certbot certonly --manual \
     --preferred-challenges=dns  \
     --email ciberkleid@vmware.com  \
     --server https://acme-v02.api.letsencrypt.org/directory \
     --agree-tos  \
     -d "*.$CLUSTER.tap-demo.coraiberkleid.net"
```

2. Create/update DNS TXT record
Create or Update GOOGLE DNS TT record as appropriate. Make sure it propagated by checking on https://toolbox.googleapps.com/apps/dig/#TXT, e.g. https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.view.tap-demo.coraiberkleid.net.

3. Create secret in cluster (SOPS file)
```
sudo kubectl create secret tls tls -n contour-tls --cert=/etc/letsencrypt/live/$CLUSTER.tap-demo.coraiberkleid.net/fullchain.pem --key=/etc/letsencrypt/live/$CLUSTER.tap-demo.coraiberkleid.net/privkey.pem --dry-run=client -o yaml > ~/.secrets/tap-gitops/gke-tap-demo/$CLUSTER-tls-secret.yaml

source ~/.secrets/setenv-sops.sh

sops --encrypt ~/.secrets/tap-gitops/gke-tap-demo/$CLUSTER-tls-secret.yaml > ~/workspace/ciberkleid-tap/tap-installer/clusters/gke-tap-$CLUSTER/cluster-config/config/custom/tls-secret.sops.yaml
```

4. In each cluster, create a namespace in `cluster-config/config/custom/contour-tls-namespace.yaml`:
```
apiVersion: v1
kind: Namespace
metadata:
  name: contour-tls
```

5. Make sure all tap-values.yaml are configured as follows (each pointing to its own domain):
```
...
    shared:
      ingress_domain: "view.tap-demo.coraiberkleid.net"
      ingress_issuer: ""
```

6. Add tls configuration to cnrs on runtime clusters and accelerator and appliveview on view cluster

