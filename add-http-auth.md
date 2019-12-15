This snippet has code examples required to setup HTTP Auth provider.

1. _Create or update your flat file with a user name and hashed password_

``` bash

    [kni@rhhi-node-worker-0 ~]$ htpasswd -c -B -b /home/kni/users.htpasswd user1 user1
    Adding password for user user1
    [kni@rhhi-node-worker-0 ~]$ htpasswd -B -b /home/kni/users.htpasswd user2 user2
    Adding password for user user2
    [kni@rhhi-node-worker-0 ~]$ htpasswd -B -b /home/kni/users.htpasswd user3 user3
    Adding password for user user3
    [kni@rhhi-node-worker-0 ~]$ cat /home/kni/users.htpasswd
    user1:$2y$05$Bu3TrBV/106D9ww6wK0DdeplKoXijwbg.Ev0uZ4q2mKf.m6ygcXwq
    user2:$2y$05$Mhs1NpfHnI3KxPxXAJbKYukWNStoDEoGF8l59Yy8S8IEPz51qTxIS
    user3:$2y$05$Pfp6OnrH9z/Hvqw0KvvLcey3QzsNVjPJ1U4SaEXl0Hs2jU5LzBK0m
    [kni@rhhi-node-worker-0 ~]$

```

2. _Create *HTPasswd* Secret_

``` bash

    oc create secret generic htpass-secret \
        --from-file=htpasswd=/home/kni/users.htpasswd \
        -n openshift-config
    secret/htpass-secret created

```

3. _Configure HTPasswd Custom Resource_

``` yaml

    cat htpasswd_identity_provider.yml

    apiVersion: config.openshift.io/v1
    kind: OAuth
    metadata:
      name: cluster
    spec:
      identityProviders:
      - name: my_htpasswd_provider
        mappingMethod: claim
        type: HTPasswd
        htpasswd:
          fileData:
            name: htpass-secret

```

4. _Add HTPasswd Identity Provider_

``` bash

    oc apply -f htpasswd_identity_provider.yml
    Warning: oc apply should be used on resource created by either oc create --save-config or oc apply
    oauth.config.openshift.io/cluster configured

```

5. _Verify Authentication_

``` bash

    [kni@rhhi-node-worker-0 ~]$ oc whoami
    system:admin
    [kni@rhhi-node-worker-0 ~]$ oc login
    Authentication required for https://api.rhhi-yp-tlv.qe.lab.redhat.com:6443 (openshift)
    Username: admin
    Password:
    Login successful.

    You don't have any projects. You can try to create a new project, by running

        oc new-project <projectname>

    [kni@rhhi-node-worker-0 ~]$ oc whoami
    admin
    [kni@rhhi-node-worker-0 ~]$

```
