projectDir := $(realpath $(dir $(firstword $(MAKEFILE_LIST))))
os := $(shell uname)
image_tag = $(VERSION)
image_name = go-reference-app
tenant_name = reference-applications

FAST_FEEDBACK_PATH = fast-feedback
EXTENDED_TEST_PATH = extended-test
PROD_PATH = prod

MONITORING=false
DASHBOARDING=false
RUN_EXTENDED_TEST=false

extended_test_replicas = 0 # until we have autoscaling working
extended_test_wait_timeout = 15m

logs_start = "$(shell date +%s)"
current_timestamp = $$(date +%s)
logs_url = "https://grafana.$(INTERNAL_SERVICES_DOMAIN)/explore?orgId=1&left=%7B%22datasource%22:%22CloudLogging%22,%22queries%22:%5B%7B%22refId%22:%22A%22,%22queryText%22:%22resource.type%3D%5C%22k8s_container%5C%22%5Cnresource.labels.namespace_name%3D%5C%22$(namespace)%5C%22%5Cnresource.labels.pod_name%3D%5C%22$(1)%5C%22%22,%22projectId%22:%22$(PROJECT_ID)%22,%22bucketId%22:%22global%2Fbuckets%2F_Default%22,%22viewId%22:%22_AllLogs%22%7D%5D,%22range%22:%7B%22from%22:%22$(logs_start)000%22,%22to%22:%22$(2)000%22%7D%7D"

.PHONY: help-p2p
help-p2p:
	@grep -E '^[a-zA-Z1-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) | grep p2p | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

.PHONY: help-all
help-all:
	@grep -E '^[a-zA-Z1-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

# P2P tasks

.PHONY: p2p-image
p2p-image:
	@echo $(image_name)

.PHONY: p2p-build
p2p-build: lint service-build service-push ## Builds the service image and pushes it to the registry

.PHONY: lint
lint: ## Linting go code
	docker run -v "$$(pwd)":/var/app ghcr.io/mgechev/revive:v1.3.2  -config /var/app/revive.toml -formatter stylish  ./var/app/cmd/service ./var/app/cmd/handler

.PHONY: service-build
service-build:
	docker build --file Dockerfile --tag $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name):$(image_tag) .
	
.PHONY: service-push
service-push: ## Push the service image
	docker image push $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name):$(image_tag)

p2p-build: lint 
	docker build --file Dockerfile --tag $(REGISTRY)/$(image_name):$(image_tag) .

.PHONY: build-functional
build-functional: ## Build the functional reference application, and package as a Docker image
	docker build --file functional/Dockerfile --tag $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name)-functional:$(image_tag) .

.PHONY: build-nft
build-nft: ## Build the non-functional-tests reference application, and package as a Docker image
	docker build --file nft/Dockerfile --tag $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name)-nft:$(image_tag) .

.PHONY: build-extended
build-extended:
	docker build --file extended/Dockerfile --tag $(REGISTRY)/$(EXTENDED_TEST_PATH)/$(image_name)-extended:$(image_tag) .

.PHONY: push-functional
push-functional: ## Push the Functional Docker image to an image registry
	docker image push $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name)-functional:$(image_tag)

.PHONY: push-nft
push-nft: ## Push the NFT Docker image to an image registry
	docker image push $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name)-nft:$(image_tag)

.PHONY: push-extended
push-extended:
	docker image push $(REGISTRY)/$(EXTENDED_TEST_PATH)/$(image_name)-extended:$(image_tag)


.PHONY: p2p-promote-generic
p2p-promote-generic:  ## Generic promote functionality
	@echo "$(red) Retagging version ${image_tag} from $(SOURCE_REGISTRY) to $(REGISTRY)"
	export CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE=$(SOURCE_AUTH_OVERRIDE) ; \
	gcloud auth configure-docker --quiet europe-west2-docker.pkg.dev; \
	docker pull $(SOURCE_REGISTRY)/$(source_repo_path)/$(image_name):${image_tag} ; \
	docker tag $(SOURCE_REGISTRY)/$(source_repo_path)/$(image_name):${image_tag} $(REGISTRY)/$(dest_repo_path)/$(image_name):${image_tag}
	@echo "$(red) Pushing version ${image_tag}"
	export CLOUDSDK_AUTH_CREDENTIAL_FILE_OVERRIDE=$(DEST_AUTH_OVERRIDE) ; \
	docker push $(REGISTRY)/$(dest_repo_path)/$(image_name):${image_tag}

.PHONY: p2p-promote-to-extended-test
p2p-promote-to-extended-test: source_repo_path=$(FAST_FEEDBACK_PATH)
p2p-promote-to-extended-test: dest_repo_path=$(EXTENDED_TEST_PATH)
p2p-promote-to-extended-test:  p2p-promote-generic

.PHONY: p2p-promote-to-prod
p2p-promote-to-prod:  source_repo_path=$(EXTENDED_TEST_PATH)
p2p-promote-to-prod:  dest_repo_path=$(PROD_PATH)
p2p-promote-to-prod:  p2p-promote-generic
