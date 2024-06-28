projectDir := $(realpath $(dir $(firstword $(MAKEFILE_LIST))))
os := $(shell uname)
image_tag = $(VERSION)
image_name = go-reference-app

FAST_FEEDBACK_PATH = fast-feedback
EXTENDED_TEST_PATH = extended-test
PROD_PATH = prod

NETWORK_NAME = $(image_name)
SERVICE_CONTAINER_NAME = service-container

.PHONY: help-p2p
help-p2p:
	@grep -E '^[a-zA-Z1-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) | grep p2p | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

.PHONY: help-all
help-all:
	@grep -E '^[a-zA-Z1-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

# P2P tasks

.PHONY: p2p-build
p2p-build: lint service-build service-push ## Builds the service image and pushes it to the registry

.PHONY: p2p-functional
p2p-functional: build-functional push-functional p2p-prod ## Runs functional tests
	docker run --network $(NETWORK_NAME) -e SERVICE_ENDPOINT=http://$(SERVICE_CONTAINER_NAME):8080 -e INGRESS_ENDPOINT=http://$(SERVICE_CONTAINER_NAME):8080 $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name)-functional:$(image_tag)

.PHONY: p2p-nft
p2p-nft: build-nft push-nft p2p-prod ## Runs non functional tests
	docker run --network $(NETWORK_NAME) -e SERVICE_ENDPOINT=http://$(SERVICE_CONTAINER_NAME):8080 -e INGRESS_ENDPOINT=http://$(SERVICE_CONTAINER_NAME):8080 $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name)-nft:$(image_tag) run ./load-testing/hello.js

.PHONY: p2p-extended-test
p2p-extended-test: p2p-prod  ## Runs extended tests
	docker run --network $(NETWORK_NAME) -e SERVICE_ENDPOINT=http://$(SERVICE_CONTAINER_NAME):8080 -e INGRESS_ENDPOINT=http://$(SERVICE_CONTAINER_NAME):8080 $(REGISTRY)/$(EXTENDED_TEST_PATH)/$(image_name)-extended:$(image_tag) run ./load-testing/hello.js

.PHONY: p2p-prod
p2p-prod: service-run ## Runs the service

## Internal Tasks, not part of the P2P
.PHONY: lint
lint: ## Linting go code
	docker run -v "$$(pwd)":/var/app ghcr.io/mgechev/revive:v1.3.2  -config /var/app/revive.toml -formatter stylish  ./var/app/cmd/service ./var/app/cmd/handler

.PHONY: service-build
service-build:
	docker build --file Dockerfile --tag $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name):$(image_tag) .

.PHONY: service-push
service-push: ## Push the service image
	docker image push $(REGISTRY)/$(FAST_FEEDBACK_PATH)/$(image_name):$(image_tag)

.PHONY: service-run
service-run: create-network
	docker run --name $(SERVICE_CONTAINER_NAME) --network $(NETWORK_NAME) -p 8080:8080 -d $(REGISTRY)/$(image_name):$(image_tag) || true

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

.PHONY: create-network
create-network:
	docker network create $(NETWORK_NAME) || true

.PHONY: p2p-promote-to-extended-test
p2p-promote-to-extended-test: source_repo_path=$(FAST_FEEDBACK_PATH)
p2p-promote-to-extended-test: dest_repo_path=$(EXTENDED_TEST_PATH)
p2p-promote-to-extended-test:  p2p-promote-generic

.PHONY: p2p-promote-to-prod
p2p-promote-to-prod:  source_repo_path=$(EXTENDED_TEST_PATH)
p2p-promote-to-prod:  dest_repo_path=$(PROD_PATH)
p2p-promote-to-prod:  p2p-promote-generic
