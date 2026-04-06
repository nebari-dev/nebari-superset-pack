# dev/Makefile
# Local development environment for nebari-superset-pack.
# Creates a kind cluster with the full Nebari stack for testing.
#
# Prerequisites: kind, kubectl, helm, docker, git
#
# Quick start:
#   make up          # Full Nebari stack + Superset
#   make up-standalone  # Standalone Superset (no Nebari)
#   make down        # Tear down

CLUSTER_NAME ?= superset-dev
NAMESPACE ?= superset
HOSTNAME ?= superset.localhost
CHART_DIR := $(shell cd .. && pwd)/chart
OPERATOR_REPO := /tmp/nebari-operator
OPERATOR_REF ?= main

.PHONY: cluster deps label-ns deploy deploy-standalone update-hosts test logs down up up-standalone

# Create kind cluster
cluster:
	@echo "==> Creating kind cluster '$(CLUSTER_NAME)'..."
	kind create cluster --name $(CLUSTER_NAME)
	@echo "==> Installing MetalLB..."
	kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
	kubectl wait --namespace metallb-system \
		--for=condition=available deployment/controller \
		--timeout=90s
	kubectl wait --namespace metallb-system \
		--for=jsonpath='{.status.numberReady}'=1 \
		daemonset/speaker \
		--timeout=90s
	$(MAKE) _metallb-config

_metallb-config:
	@METALLB_BASE_IP=$$(docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}' | tr ' ' '\n' | grep -F '.' | head -1 | cut -d. -f1,2); \
	echo "==> Configuring MetalLB IP pool ($${METALLB_BASE_IP}.255.200-$${METALLB_BASE_IP}.255.250)..."; \
	printf 'apiVersion: metallb.io/v1beta1\nkind: IPAddressPool\nmetadata:\n  name: dev-pool\n  namespace: metallb-system\nspec:\n  addresses:\n    - %s.255.200-%s.255.250\n---\napiVersion: metallb.io/v1beta1\nkind: L2Advertisement\nmetadata:\n  name: dev-l2\n  namespace: metallb-system\n' "$${METALLB_BASE_IP}" "$${METALLB_BASE_IP}" | kubectl apply -f -

# Install Nebari stack (Envoy Gateway, cert-manager, Keycloak, nebari-operator)
deps:
	@echo "==> Cloning nebari-operator..."
	@if [ ! -d "$(OPERATOR_REPO)" ]; then \
		git clone --depth 1 --branch $(OPERATOR_REF) https://github.com/nebari-dev/nebari-operator.git $(OPERATOR_REPO); \
	fi
	@echo "==> Installing services (Envoy Gateway, cert-manager, Keycloak)..."
	$(OPERATOR_REPO)/dev/scripts/services/install.sh
	@echo "==> Setting up Keycloak..."
	$(OPERATOR_REPO)/dev/scripts/services/keycloak/setup.sh
	@echo "==> Installing nebari-operator..."
	kubectl apply -f https://github.com/nebari-dev/nebari-operator/releases/download/v0.1.0-alpha.5/install.yaml
	kubectl wait --for=condition=available deployment/nebari-operator-controller-manager \
		-n nebari-operator-system --timeout=120s

# Label namespace for nebari-operator
label-ns:
	@echo "==> Creating and labeling namespace '$(NAMESPACE)'..."
	kubectl create namespace $(NAMESPACE) --dry-run=client -o yaml | kubectl apply -f -
	kubectl label namespace $(NAMESPACE) nebari.dev/managed=true --overwrite

# Deploy with Nebari integration
deploy: label-ns
	@echo "==> Building chart dependencies..."
	cd $(CHART_DIR) && helm dependency build
	@echo "==> Deploying nebari-superset..."
	helm upgrade --install superset $(CHART_DIR) \
		-n $(NAMESPACE) \
		--set nebariapp.enabled=true \
		--set nebariapp.hostname=$(HOSTNAME) \
		--set nebariapp.auth.enabled=true \
		--set 'superset.bootstrapScript=#!/bin/bash\nuv pip install authlib ".[postgres]" && if [ ! -f ~/bootstrap ]; then echo "Running Superset" > ~/bootstrap; fi' \
		--wait --timeout 10m
	@echo "==> Superset deployed with Nebari integration."

# Deploy standalone (no Nebari)
deploy-standalone:
	@echo "==> Building chart dependencies..."
	cd $(CHART_DIR) && helm dependency build
	@echo "==> Deploying nebari-superset (standalone)..."
	kubectl create namespace $(NAMESPACE) --dry-run=client -o yaml | kubectl apply -f -
	helm upgrade --install superset $(CHART_DIR) \
		-n $(NAMESPACE) \
		-f $(shell cd .. && pwd)/examples/standalone-values.yaml \
		--wait --timeout 10m
	@echo "==> Superset deployed in standalone mode."

# Update /etc/hosts
update-hosts:
	@echo "==> Updating /etc/hosts for $(HOSTNAME)..."
	@if ! grep -q "$(HOSTNAME)" /etc/hosts; then \
		echo "127.0.0.1 $(HOSTNAME)" | sudo tee -a /etc/hosts; \
	fi

# Verify deployment
test:
	@echo "==> Checking pod status..."
	kubectl get pods -n $(NAMESPACE) -l app=superset
	@echo ""
	@echo "==> Checking services..."
	kubectl get svc -n $(NAMESPACE)
	@echo ""
	@if kubectl get crd nebariapps.reconcilers.nebari.dev > /dev/null 2>&1; then \
		echo "==> Checking NebariApp status..."; \
		kubectl get nebariapp -n $(NAMESPACE); \
		echo ""; \
		echo "==> Checking HTTPRoute..."; \
		kubectl get httproute -n $(NAMESPACE); \
		echo ""; \
		echo "==> Checking SecurityPolicy..."; \
		kubectl get securitypolicy -n $(NAMESPACE) 2>/dev/null || echo "No SecurityPolicy found"; \
	fi

# Tail logs
logs:
	kubectl logs -n $(NAMESPACE) -l app=superset -f --tail=100

# Tear down
down:
	@echo "==> Deleting kind cluster '$(CLUSTER_NAME)'..."
	kind delete cluster --name $(CLUSTER_NAME)
	@rm -rf $(OPERATOR_REPO)

# Full stack: Nebari + Superset
up: cluster deps deploy update-hosts
	@echo ""
	@echo "==> nebari-superset is running!"
	@echo "    URL: https://$(HOSTNAME)"
	@echo "    Keycloak: admin / nebari-admin"
	@echo ""
	@echo "    Run 'make test' to verify the deployment."
	@echo "    Run 'make logs' to tail Superset logs."
	@echo "    Run 'make down' to tear down."

# Full stack: Standalone
up-standalone: cluster deploy-standalone
	@echo ""
	@echo "==> nebari-superset is running in standalone mode!"
	@echo "    Port-forward: kubectl port-forward svc/superset-superset 8088:8088 -n $(NAMESPACE)"
	@echo ""
	@echo "    Run 'make test' to verify the deployment."
	@echo "    Run 'make logs' to tail Superset logs."
	@echo "    Run 'make down' to tear down."
