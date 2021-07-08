CLUSTER_NAME ?= prototype-general
FILE ?= prototype-general-kubeconfig

.PHONY: secret

secret:
	kubectl create secret generic ${CLUSTER_NAME}-coredns --from-file ${FILE} -n kube-system --validate=true
