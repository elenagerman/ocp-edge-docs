Some notes wrt setting up [cucushift](https://github.com/openshift/verification-tests) env on RHEL-8 provisionhost

Clone repo
```bash
    git clone https://github.com/openshift/verification-tests.git
```

Export `GEM_HOME` to make `ruby` happy :)
```bash
    export GEM_HOME=/home/kni/.gem/ruby
```

Install dependencies on RHEL-8
```bash
    sudo yum install libxml2.x86_64 python3-libxml2.x86_64 libxml2-devel.x86_64 python3-lxml.x86_64

    sudo yum install libxslt.x86_64 libxslt-devel.x86_64 python3-lxml.x86_64 python2-lxml.x86_64

```

Continue with installation
``` bash
    cd verification-tests

    sudo tools/install_os_deps.sh

    tools/hack_bundle.rb
    ...
    Installing watir 6.16.5
    Bundle complete! 29 Gemfile dependencies, 93 gems now installed.
    Use `bundle info [gemname]` to see where a bundled gem is installed.
    Post-install message from i18n:

    HEADS UP! i18n 1.1 changed fallbacks to exclude default locale.
    But that may break your application.

    Please check your Rails app for 'config.i18n.fallbacks = true'.
    If you're using I18n (>= 1.1.0) and Rails (< 5.2.2), this should be
    'config.i18n.fallbacks = [I18n.default_locale]'.
    If not, fallbacks will be broken in your app by I18n 1.1.x.

    For more info see:
    https://github.com/svenfuchs/i18n/releases/tag/v1.1.0
```

Configure `cucushift` variables

> NOTE: for OPENSHIFT\_ENV\_OCP4\_HOSTS hostnames should match output from `oc get nodes` command
>       for e.g.: OPENSHIFT_ENV_OCP4_HOSTS=master-0.lab.redhat.com:etcd:master:worker,master-1.lab.redhat.com:etcd:master:worker,...

``` bash
    export BUSHSLICER_DEFAULT_ENVIRONMENT=ocp4
    export OPENSHIFT_ENV_OCP4_HOSTS=master-0:etcd:master:worker,master-1:etcd:master:worker,master-2:etcd:master:worker
    export OPENSHIFT_ENV_OCP4_USER_MANAGER_USERS=user1:<USER1-PASSWORD>
```

There is some bug so had to modify `config/config.yaml`

> NOTE: user1 was created before the test, [more info at](https://gitlab.cee.redhat.com/yprokule/ocp4-misc/blob/master/add-http-auth.md)

```diff
diff --git a/config/config.yaml b/config/config.yaml
index 256b7a7..d89ba6e 100644
--- a/config/config.yaml
+++ b/config/config.yaml
@@ -72,9 +72,10 @@ environments:
     api_port: 8443
   ocp4: &ocp4
     # hosts: use OPENSHIFT_ENV_OSE_HOSTS=host:role1:...,host2:role1:...
+    #hosts: master-0:etcd:master:node,master-1:etcd:master:node,master-2:etcd:master:node
     hosts_type: OCDebugAccessibleHost
     # this is the user for remote access to the OpenShift nodes
-    user: root
+    user: core
     type: StaticEnvironment
     user_manager: auto
     # set users in OPENSHIFT_ENV_OSE_USER_MANAGER_USERS=user:password,...
```

Run tests
```bash

    bundle exec cucumber features/cli/configmap.feature:6
    ...
    waiting for operation up to 3600 seconds..
      # @author chezhang@redhat.com
      # @case_id OCP-10805
      @smoke
      Scenario: Consume ConfigMap in environment variables          # features/cli/configmap.feature:6
          [12:37:04] INFO> === Before Scenario: Consume ConfigMap in environment variables ===
          [12:37:04] INFO> Shell Commands: mkdir -v -p '/home/kni/workdir/worker-1-kni'

          [12:37:04] INFO> Exit Status: 0
          [12:37:04] INFO> === End Before Scenario: Consume ConfigMap in environment variables ===
        Given I have a project                                      # features/step_definitions/project.rb:7
          [12:37:04] INFO> HTTP GET https://master-0:6443/.well-known/oauth-authorization-server
          [12:37:04] INFO> HTTP GET took 0.020 sec: 200 OK | application/json 624 bytes
      ..
          [12:37:32] INFO> Exit Status: 0
          [12:37:32] INFO> === End After Scenario: Consume ConfigMap in environment variables ===

    1 scenario (1 passed)
    13 steps (13 passed)
    0m28.766s
    [12:37:32] INFO> === At Exit ===
```
