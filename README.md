# Patching Packages with >0.0.0 version

```
Patching Packages with different versions
============================================
source tanzucli.src
export KUBECONFIG="/root/.config/tanzu/kube/config"
export proj="AMER-East"
export sp="orfspace1"
export org="sa-tanzu-platform"
export cl="orfclustergroup"
export w=''
#export w='--wide'
export line="-----------------------------------------------------------------"
yes | tanzu context delete $org
source ./tanzucli.src
tanzu login
tanzu project use $proj
tanzu operations clustergroup use  $cl

#Before Picture: 

for package in $(kubectl get pkgi -o json | jq -r '.items[].metadata.name'); do kubectl get pkgi $package -o json | jq -r '.| "\(.metadata.name) \(.spec.packageRef.versionSelection.constraints)" '; done;


Patch: 

for package in $(kubectl get pkgi -o json | jq -r '.items[].metadata.name'); do kubectl patch packageinstalls.packaging.carvel.dev $package  --type='json' -p '[{"op": "replace", "path": "/spec/packageRef/versionSelection/constraints", "value":">0.0.0"}]'; done;

After Patch Picture:

for package in $(kubectl get pkgi -o json | jq -r '.items[].metadata.name'); do kubectl get pkgi $package -o json | jq -r '.| "\(.metadata.name) \(.spec.packageRef.versionSelection.constraints)" '; done;


```


# My Space was yellow do to packages version issues 
# A re-schedule of the space placed both nginx on the same cluster
# Creating to avaliability targets and the resulting scheduling problem
# Turns out label issues

```
#Does the space have an issue? 
tanzu space list
tanzu space get orfspace1 
#Cluster group
tanzu project use AMER-East
tanzu operations clustergroup use orfclustergroup
tanzu operations clustergroup get orfclustergroup
#back to project
tanzu project use AMER-East
 k get space orfspace1
# found issue
k get space orfspace1 -o yaml | grep message
#    message: Referred Profiles have been successfully resolved
#   message: Space has Minimum available replicas ready
#    message: ManagedNamespaceSet "orfspace1-5657bdb868" has successfully progressed.
#    message: AvailabilityTarget orfcluster1target79 is not ready
#    message: Error while resolving Referenced resources
#    message: Error while resolving Referenced resources
#
# Running just on one cluster
#
 k get  mn | grep orf
# orfspace1-5657bdb868-m48fd                34m     orfcluster2target79   orfscluster2                                                                      default
# orfspace1-5657bdb868-dzjg2                31m     orfcluster1target79                                                                                     
#
# look at every space 
 k get mn orfspace1-5657bdb868-dzjg2   
# NAME                         AGE   AVAILABILITY_TARGET   CLUSTER_NAME   CLUSTER_NAMESPACE
# orfspace1-5657bdb868-dzjg2   33m   orfcluster1target79                  
#
k get mn orfspace1-5657bdb868-m48fd
# NAME                         AGE   AVAILABILITY_TARGET   CLUSTER_NAME   CLUSTER_NAMESPACE
# orfspace1-5657bdb868-m48fd   36m   orfcluster2target79   orfscluster2   default
#
k get mn orfspace1-5657bdb868-m48fd   -o yaml | grep message
#    message: cluster/s satisfies AvailabilityTarget rules
#    message: cluster/s satisfies Capabilities
#    message: found a matching cluster
#    message: Traits Installed
#    message: Ready
#
k get mn orfspace1-5657bdb868-dzjg2   -o yaml | grep message
#    message: no clusters are available in the Availability Target
#    message: no clusters are available in the Availability Target
#    message: no clusters are available in the Availability Target
tanzu operations clustergroup use orfclustergroup      
#
# both clusters look good
#
 k get kubernetescluster orfscluster2 -o yaml | grep message
#    message: cluster is attached and healthy
k get kubernetescluster orfscluster -o yaml | grep message
#    message: cluster is attached and healthy
#
# Targets are the issue
tanzu project use AMER-East
# get all of them
k get avt -A
#
# Getting hot here.. 
#
k get avt -A | grep orf
#default     orfcluster2target79             67m    True    clusters selected based on provided rules
#default     orfstarget79                    34d    False   no clusters match provided rules
#default     orfcluster1target79             68m    False   no clusters match provided rules
#
# look at the target selection
#
k get avt orfcluster2target79 -o yaml | yq -r ".spec.affinity.clusterAffinity.clusterSelectorTerms"
#    "matchExpressions": [
#      {
#        "key": "mycluster2",
#        "operator": "Exists"
#      }
#
k get avt orfcluster1target79 -o yaml | yq -r ".spec.affinity.clusterAffinity.clusterSelectorTerms"
#    "matchExpressions": [
#      {
#        "key": "mycluster1",
#        "operator": "Exists"
#      }
#
#lets look at the clusters and their labels
#
tanzu operations clustergroup use orfclustergroup
#
k get kubernetesclusters                       
# orfscluster2
# orfscluster
#
k get kubernetesclusters       orfscluster2 -o yaml | yq -r ".metadata.labels"
#{
#  "claimed.internal.apis.kcp.io/1c8nAiAmP283rsP5PQqYqmhbwN8cQXbC3Q9L0R": "9V6wt3DAYBEuLIcXFkXhEDhAA42xt1UagdyBX0",
#  "mycluster2": "orfscluster2",
#  "tmc.cloud.vmware.com/creator": "ogelbrich",
#  "topology.kubernetes.io/region": ""
#}
k get kubernetesclusters       orfscluster -o yaml | yq -r ".metadata.labels"
#{
#  "claimed.internal.apis.kcp.io/1c8nAiAmP283rsP5PQqYqmhbwN8cQXbC3Q9L0R": "9V6wt3DAYBEuLIcXFkXhEDhAA42xt1UagdyBX0",
#  "myclsuter1": "orfcluster1",
#  "tmc.cloud.vmware.com/creator": "ogelbrich",
#  "topology.kubernetes.io/region": ""
#}
# Bingo I can’t spell mycluster1—>>> myclsuter1
# Fixed label in GUI 
#
# back to normal a nginx running on each cluster
#
k config use-context orfscluster; k get pods -A | grep nginx ; k config use-context orfscluster2; k get pods -A | grep nginx
#
# Switched to context "orfscluster".
# orfspace1-5657bdb868-dzjg2     orf-nginx-app-engine-7cc88c4dc7-24mr7                             2/2     Running            0                118s
#
# Switched to context "orfscluster2".
# orfspace1-5657bdb868-m48fd     orf-nginx-app-engine-57c7c88c69-ll28m                             2/2     Running     0                85m

```
