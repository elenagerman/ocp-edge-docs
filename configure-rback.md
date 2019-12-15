1. _Assign *cluster-admin* cluster role to user *admin*_

``` bash

    [kni@rhhi-node-worker-0 ~]$ oc whoami
    kube:admin
    [kni@rhhi-node-worker-0 ~]$ oc adm policy add-cluster-role-to-user cluster-admin admin
    clusterrole.rbac.authorization.k8s.io/cluster-admin added: "admin"

```

2. _Create user projects_

``` bash
    [kni@rhhi-node-worker-0 ~]$ oc new-project user1-project
    Now using project "user1-project" on server "https://api.rhhi-yp-tlv.qe.lab.redhat.com:6443".

    You can add applications to this project with the 'new-app' command. For example, try:

        oc new-app django-psql-example

    to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

        kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

    [kni@rhhi-node-worker-0 ~]$ oc new-project user2-project
    Now using project "user2-project" on server "https://api.rhhi-yp-tlv.qe.lab.redhat.com:6443".

    You can add applications to this project with the 'new-app' command. For example, try:

        oc new-app django-psql-example

    to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

        kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

    [kni@rhhi-node-worker-0 ~]$ oc new-project user3-project
    Now using project "user3-project" on server "https://api.rhhi-yp-tlv.qe.lab.redhat.com:6443".

    You can add applications to this project with the 'new-app' command. For example, try:

        oc new-app django-psql-example

    to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

        kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```

3. _Assign users as admins in projects_

``` bash

    [kni@rhhi-node-worker-0 ~]$ oc adm policy add-role-to-user admin user1 -n user1-project
    Warning: User 'user1' not found
    clusterrole.rbac.authorization.k8s.io/admin added: "user1"

    [kni@rhhi-node-worker-0 ~]$ oc adm policy add-role-to-user admin user2 -n user2-project
    Warning: User 'user2' not found
    clusterrole.rbac.authorization.k8s.io/admin added: "user2"

    [kni@rhhi-node-worker-0 ~]$ oc adm policy add-role-to-user admin user3 -n user3-project
    Warning: User 'user3' not found
    clusterrole.rbac.authorization.k8s.io/admin added: "user3"

```

4. _Verify user can access own projects_

``` bash

    [kni@rhhi-node-worker-0 ~]$ oc login
    Authentication required for https://api.rhhi-yp-tlv.qe.lab.redhat.com:6443 (openshift)
    Username: user1
    Password:
    Login successful.

    You have one project on this server: "user1-project"

    Using project "user1-project".
    [kni@rhhi-node-worker-0 ~]$ oc get nodes
    Error from server (Forbidden): nodes is forbidden: User "user1" cannot list resource "nodes" in API group "" at the cluster scope
    [kni@rhhi-node-worker-0 ~]$ oc get all
    No resources found in user1-project namespace.
    [kni@rhhi-node-worker-0 ~]$ oc status
    In project user1-project on server https://api.rhhi-yp-tlv.qe.lab.redhat.com:6443

    You have no services, deployment configs, or build configs.
    Run 'oc new-app' to create an application.
    [kni@rhhi-node-worker-0 ~]$ oc login -u user2
    Authentication required for https://api.rhhi-yp-tlv.qe.lab.redhat.com:6443 (openshift)
    Username: user2
    Password:
    Login successful.

    You have one project on this server: "user2-project"

    Using project "user2-project".
    [kni@rhhi-node-worker-0 ~]$ oc get all
    No resources found in user2-project namespace.

```

5. _Verify users cannot access other namespaces_

``` bash

    [kni@rhhi-node-worker-0 ~]$ oc whoami
    user2

    [kni@rhhi-node-worker-0 ~]$ oc get all -n user1-project
    Error from server (Forbidden): pods is forbidden: User "user2" cannot list resource "pods" in API group "" in the namespace "user1-project"
    Error from server (Forbidden): replicationcontrollers is forbidden: User "user2" cannot list resource "replicationcontrollers" in API group "" in the namespace "user1-project"
    Error from server (Forbidden): services is forbidden: User "user2" cannot list resource "services" in API group "" in the namespace "user1-project"
    Error from server (Forbidden): daemonsets.apps is forbidden: User "user2" cannot list resource "daemonsets" in API group "apps" in the namespace "user1-project"
    Error from server (Forbidden): deployments.apps is forbidden: User "user2" cannot list resource "deployments" in API group "apps" in the namespace "user1-project"
    Error from server (Forbidden): replicasets.apps is forbidden: User "user2" cannot list resource "replicasets" in API group "apps" in the namespace "user1-project"
    Error from server (Forbidden): statefulsets.apps is forbidden: User "user2" cannot list resource "statefulsets" in API group "apps" in the namespace "user1-project"
    Error from server (Forbidden): horizontalpodautoscalers.autoscaling is forbidden: User "user2" cannot list resource "horizontalpodautoscalers" in API group "autoscaling" in the namespace "user1-project"
    Error from server (Forbidden): jobs.batch is forbidden: User "user2" cannot list resource "jobs" in API group "batch" in the namespace "user1-project"
    Error from server (Forbidden): cronjobs.batch is forbidden: User "user2" cannot list resource "cronjobs" in API group "batch" in the namespace "user1-project"
    Error from server (Forbidden): deploymentconfigs.apps.openshift.io is forbidden: User "user2" cannot list resource "deploymentconfigs" in API group "apps.openshift.io" in the namespace "user1-project"
    Error from server (Forbidden): buildconfigs.build.openshift.io is forbidden: User "user2" cannot list resource "buildconfigs" in API group "build.openshift.io" in the namespace "user1-project"
    Error from server (Forbidden): builds.build.openshift.io is forbidden: User "user2" cannot list resource "builds" in API group "build.openshift.io" in the namespace "user1-project"
    Error from server (Forbidden): imagestreams.image.openshift.io is forbidden: User "user2" cannot list resource "imagestreams" in API group "image.openshift.io" in the namespace "user1-project"
    Error from server (Forbidden): routes.route.openshift.io is forbidden: User "user2" cannot list resource "routes" in API group "route.openshift.io" in the namespace "user1-project"

```
