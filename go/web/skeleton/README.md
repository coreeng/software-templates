# CECG IDP Reference Application - GoLang

Reference Golang application for the CECG Developer Platform.

# Path to Production

The P2P uses Github Actions to interact with the platform.
There are 3 variables you need to set in order for this to work. On the repositories' web page, navigate to `Settings -> Environments` and ensure you have `gcp-dev` and `gcp-prod` environments created.
For each of these, you'll need these variables:
- BASE_DOMAIN: The base domain configured on the deployment, prefixed by the environment (eg. `gcp-dev.cecg.platform.cecg.io`)
- DPLATFORM: As defined in your developer platform environments file e.g. `environment: "gcp-dev"`
- PROJECT_ID: The GCP Id of the project. You can see this in the GCP console page.
- PROJECT_NUMBER: Similar to project id, this is the numeric values for the GCP project. You can see this value in GCP console next to the project id.

In addition, you need to make sure that you have a repository environment variables with:
- TENANT_NAME: Name of your tenant. There is a namespace created with the same name as your tenant and that will be used as a parent namespace.
- FAST_FEEDBACK={"include": [{"deploy_env": "gcp-dev"}]} - Defines which environment the fast feedback pipeline runs on
- EXTENDED_TEST={"include": [{"deploy_env": "gcp-dev"}]} -  Defines which environment the extended testspipeline runs on
- PROD={"include": [{"deploy_env": "gcp-prod"}]} -  Defines which environment the production pipeline runs on

As part of the P2P ,using Hierarchical Namespace Controller, 2 child namespaces will be created:
- <tenant-name>-functional
- <tenant-name>-nft

The application is deployed to each on of this following the shape:
| Build Service | -> | Functional testing | -> | NF testing | -> | Promote image to Extended tests |


The tests are execute as helm tests. For that to work, each test phase is packaged in a docker image and pushed to a registry. It's then execute after the deployment of the respective environment to ensure the service is working correctly.

## Functional Testing

Stubbed Functional Tests using [Cucumber Godog](https://github.com/cucumber/godog)

This namespace is used to test the functionality of the app. Currently using BDD (Behaviour driven development)

## NFT

Load tests using [K6](https://k6.io/).

This namespace is used to test how the service behaves under load. It asserts response time percentiles and ensure there are no errors


## Requirements

The interface between the P2P and the application is `Make`. 
For everything to your for you locally you need to ensure you have the following tools installed on your machine:
* Make
* Docker
* Kubectl
* Helm

## P2P interface

The P2P uses the Makefile to deploy and test the application. It expects these tasks to exist:
* p2p-build - Builds the service image and pushes it to the registry
* p2p-functional - Runs only functional helm tests
* p2p-nft -Runs only NFT helm tests
* p2p-promote-to-extended-test - Promotes the images when running on main branch and both NFT and Functional steps are successful 
You can run `make help-p2p` to list the available p2p functions or `help-all` to see all available functions.

###  Prerequisites for local run

* GCloud login
* GCloud registry login


#### Push the image

There's a shared tenant registry created `europe-west2-docker.pkg.dev/<project_id>/tenant`. You'll need to set your project_id and export this string as an environment variable called `REGISTRY`, for example:
```
export REGISTRY=europe-west2-docker.pkg.dev/MY_PROJECT_ID/tenant
```

#### Ingress URL construction

For ingress to be configured correctly you'll need to set up the environment that you want to deploy to, as well as the base url to be used. 
This must match one of the `ingress_domains` configured for that environment. For example, inside CECG we have an environment called `gcp-dev` that's ingress domian is set to `gcp-dev.cecg.platform.cecg.io`.

This reference app assumes `<environment>.<domain>`, check with your deployment of the CECG developer platform if this is the case.

This wil construct the base URL as `<environment>.<domain>`, for example, `gcp-dev.cecg.platform.cecg.io`.

```
export BASE_DOMAIN=gcp-dev.cecg.platform.cecg.io 
```
