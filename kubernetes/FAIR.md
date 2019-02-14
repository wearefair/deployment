# Fair Changes
Using the custom-compiled version of [CoreDNS with Kubernetai](https://github.com/wearefair/kubernetai/pull/1/files), we're setting up a cross-cluster Corefile. This allows CoreDNS to talk to both general and secure K8 API servers in order to resolve DNS.

## The Theory
Kubernetai's syntax is the same as the [Kubernetes](https://coredns.io/plugins/kubernetes/) plugin. This means we should be able to declare a Corefile that resolves local DNS and then falls through to the next Kubernetes cluster.

A sample Corefile for the prototype cluster would look like this:
```
.:53 {
    errors
    health
    kubernetai cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      upstream
      fallthrough in-addr.arpa ip6.arpa
    }
    kubernetai cluster.local in-addr.arpa ip6.arpa {
      endpoint https://prototype-k8.sandbox-secure.fair.engineering
      namespaces default
      kubeconfig /root/.kubeconfig prototype-secure
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

Essentially, if DNS for the original K8 cluster (local cluster) fails, then it should attempt to resolve DNS against the remote cluster.

## Components Required
* Custom compiled Kubernetai image
* CoreDNS service token from the "remote" cluster
* Kubeconfig with the service token
* Updated Corefile that points to the remote cluster (as seen above)

## General Steps
These are general steps for how I got to this point so I can backtrack in case I lose context on this

0. Compile and build custom version of [Kubernetai Docker image](https://github.com/wearefair/kubernetai/pull/1/files)
1. Swap to the Kubernetes context to create the CoreDNS pods in.
2. Run `deploy.sh`. It's been modified to pipe all contents to `rendered.yaml`
3. Apply and create the CoreDNS service account stuff to generate the tokens. Do this for both sides.
4. Grab the service account tokens for both CoreDNS service accounts
5. Create Kubeconfigs for the service accounts
6. Create secrets from the Kubeconfigs (sample of how to do it is in the `secret` invocation in the Makefile)
7. Update the `rendered.yaml` with a Corefile structured similarly to the above, with a secret volume for the kubeconfig, and with the custom compiled CoreDNS image
8. `kubectl apply -f rendered.yaml`. This should swap out the existing KubeDNS pods with the CoreDNS ones. Tail the logs to make sure it doesn't blow up.
