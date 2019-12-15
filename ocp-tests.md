## OCP Tests

### OCP Tests Installation

* Clone `origin` repo

``` bash
    cd /var/tmp
    git clone https://github.com/openshift/origin -b release-4.3
```

* Download and install golang binary >= 1.12

``` bash
    cd /var/tmp
    curl -4kv -O https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
    tar xzvf go1.12.7.linux-amd64.tar.gz
```

* Build OCP tests

``` bash
    cd /var/tmp/origin
    make WHAT=cmd/openshift-tests
```

### OCP tests invocation

_`NOTE: Assumed that GO is unpacked under /var/tmp/go directory. Adjust according to your setup`_

``` bash
    KUBECONFIG=rhhi-qe-core/kubeconfig GOPATH=/var/tmp/go/ /var/tmp/origin/_output/local/bin/linux/amd64/openshift-tests help run
```

Below command invokes `openshift/conformance/parallel` test suite agains RHHI/OCP environment

``` bash
    KUBECONFIG=rhhi-qe-core/kubeconfig GOPATH=/var/tmp/go/ \
        /var/tmp/origin/_output/local/bin/linux/amd64/openshift-tests run \
        openshift/conformance/parallel -o ocp-carbon-parallel.log
```

### List of OCP test suites

*    **openshift/conformance**
      Tests that ensure an OpenShift cluster and components are working properly.

*    **openshift/conformance/parallel**
      Only the portion of the openshift/conformance test suite that run in parallel.

*    **openshift/conformance/serial**
      Only the portion of the openshift/conformance test suite that run serially.

*    **openshift/conformance/multitenant**
      Only the portion of the openshift/conformance test suite that applies to the openshift-sdn multitenant plugin.

*    **openshift/disruptive**
      The disruptive test suite.

*    **kubernetes/conformance**
      The default Kubernetes conformance suite.

*    **openshift/build**
      Tests that exercise the OpenShift build functionality.

*    **openshift/image-registry**
      Tests that exercise the OpenShift image-registry functionality.

*    **openshift/image-ecosystem**
      Tests that exercise language and tooling images shipped as part of OpenShift.

*    **openshift/jenkins-e2e**
      Tests that exercise the OpenShift / Jenkins integrations provided by the OpenShift Jenkins image/plugins and the Pipeline Build Strategy.

*    **openshift/scalability**
      Tests that verify the scalability characteristics of the cluster. Currently this is focused on core performance behaviors and preventing regressions.

*    **openshift/conformance-excluded**
      Run only tests that are excluded from conformance. Makes identifying omitted tests easier.

*    **openshift/test-cmd**
      Run only tests for test-cmd.

*    **all**
      Run all tests.
