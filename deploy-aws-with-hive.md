# Install hive

## run the following steps in order to deploy hive:

```
# go version > 1.12 is one of the prerequisites
sudo yum install golang-bin-1.12.8-2.module+el8.1.0+4089+be929cf8 -y

#kustomize is a prerequisite
sudo yum install -y wget
wget -O /tmp/kustomize.tgz https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv3.3.0/kustomize_v3.3.0_linux_amd64.tar.gz
tar xzvf /tmp/kustomize.tgz
sudo mv ./kustomize /usr/sbin/

#must operate from a specific directory
mkdir -p ~/go/src/github.com/openshift
cd ~/go/src/github.com/openshift

#clone the hive repo
git clone https://github.com/openshift/hive.git
cd hive

#mockgen binary is required
go get github.com/golang/mock/mockgen
export PATH=~/go/bin:$PATH

#The following vars should be initialized (in general goroot is where your go install is, gopath is where you work on code):
export GOROOT=/usr/lib/golang
export GOPATH=~/go

#Deploy hive
if [ -f ~/dev-scripts/ocp/auth/kubeconfig ]; then
    export KUBECONFIG=~/dev-scripts/ocp/auth/kubeconfig
else
    export KUBECONFIG=~/clusterconfigs/auth/kubeconfig
fi
make deploy

#At this point you should be able to see pods starting/running with:
oc get pod -n hive

#Optional install hiveutil - this will allow creating clusters and/or cluster configuration yaml with hiveutil
make hiveutil
```


## To delete Hive, run the following command:
```
oc get deployment -n hive -o name |xargs -i oc -n hive delete {}
```
