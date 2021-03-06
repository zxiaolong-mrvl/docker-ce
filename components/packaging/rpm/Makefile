include ../containerd.mk

ARCH=$(shell uname -m)
ENGINE_DIR:=$(CURDIR)/../../engine
CLI_DIR:=$(CURDIR)/../../cli
GITCOMMIT=$(shell cd $(ENGINE_DIR) && git rev-parse --short HEAD)
VERSION?=0.0.0-dev
GO_BASE_IMAGE=golang
GO_VERSION:=1.10.4
GO_IMAGE?=$(GO_BASE_IMAGE):$(GO_VERSION)
GEN_RPM_VER=$(shell ./gen-rpm-ver $(CLI_DIR) $(VERSION))
CHOWN=docker run --rm -i -v $(CURDIR):/v -w /v alpine chown

DOCKERFILE=Dockerfile
ifdef NEEDS_ARCH_SPECIFIC
	DOCKERFILE=Dockerfile.$(ARCH)
endif
BUILD?=docker build --build-arg GO_IMAGE=$(GO_IMAGE) -t rpmbuild-$@/$(ARCH) -f $@/$(DOCKERFILE) .

SPEC_FILES?=docker-ce.spec docker-ce-cli.spec
SPECS?=$(addprefix SPECS/, $(SPEC_FILES))
RPMBUILD=docker run --privileged --rm -i\
	-v $(CURDIR)/rpmbuild/SOURCES:/root/rpmbuild/SOURCES \
	-v $(CURDIR)/rpmbuild/RPMS:/root/rpmbuild/RPMS \
	-v $(CURDIR)/rpmbuild/SRPMS:/root/rpmbuild/SRPMS
RPMBUILD_FLAGS?=-ba\
	--define '_gitcommit $(word 3,$(GEN_RPM_VER))' \
	--define '_release $(word 2,$(GEN_RPM_VER))' \
	--define '_version $(word 1,$(GEN_RPM_VER))' \
	--define '_origversion $(word 4, $(GEN_RPM_VER))' \
	$(SPECS)
RUN?=$(RPMBUILD) rpmbuild-$@/$(ARCH) $(RPMBUILD_FLAGS)
ENGINE_IMAGE=docker/engine-community
ENGINE_SCOPE=ce

SOURCE_FILES=containerd-proxy.tgz cli.tgz containerd-shim-process.tar docker.service dockerd.json engine.tar
SOURCES=$(addprefix rpmbuild/SOURCES/, $(SOURCE_FILES))


.PHONY: help
help: ## show make targets
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {sub("\\\\n",sprintf("\n%22c"," "), $$2);printf " \033[36m%-20s\033[0m  %s\n", $$1, $$2}' $(MAKEFILE_LIST)

.PHONY: clean
clean: ## remove build artifacts
	[ ! -d rpmbuild ] || $(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild
	$(RM) -r rpmbuild/
	[ ! -d artifacts ] || $(CHOWN) -R $(shell id -u):$(shell id -g) artifacts
	$(RM) -r artifacts/
	[ ! -d tmp ] || $(CHOWN) -R $(shell id -u):$(shell id -g) tmp
	$(RM) -r tmp/
	-docker rm docker2oci

.PHONY: rpm
rpm: fedora centos ## build all rpm packages

.PHONY: fedora
fedora: fedora-28 fedora-27 fedora-26 ## build all fedora rpm packages

.PHONY: centos
centos: centos-7 ## build all centos rpm packages

.PHONY: fedora-28
fedora-28: ## build fedora-28 rpm packages
fedora-28: $(SOURCES)
	$(CHOWN) -R root:root rpmbuild
	$(BUILD)
	$(RUN)
	$(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild

.PHONY: fedora-27
fedora-27: ## build fedora-27 rpm packages
fedora-27: $(SOURCES)
	$(CHOWN) -R root:root rpmbuild
	$(BUILD)
	$(RUN)
	$(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild

.PHONY: centos-7
centos-7: ## build centos-7 rpm packages
centos-7: $(SOURCES)
	$(CHOWN) -R root:root rpmbuild
	$(BUILD)
	$(RUN)
	$(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild

rpmbuild/SOURCES/cli.tgz:
	mkdir -p rpmbuild/SOURCES
	docker run --rm -i -w /v \
		-v $(CLI_DIR):/cli \
		-v $(CURDIR)/rpmbuild/SOURCES:/v \
		alpine \
		tar -C / -c -z -f /v/cli.tgz --exclude .git cli

rpmbuild/SOURCES/containerd-proxy.tgz:
	mkdir -p tmp/
	curl -fL -o tmp/containerd-proxy.tgz "https://github.com/crosbymichael/containerd-proxy/archive/$(CONTAINERD_PROXY_COMMIT).tar.gz"
	tar xzf tmp/containerd-proxy.tgz -C tmp/
	mv tmp/containerd-proxy-$(CONTAINERD_PROXY_COMMIT) tmp/containerd-proxy
	mkdir -p $(@D)
	$(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild
	tar -zcf $@ -C tmp/ containerd-proxy
	rm -rf tmp/

rpmbuild/SOURCES/containerd-shim-process.tar:
	$(CTR) content fetch $(CONTAINERD_SHIM_PROCESS_IMAGE)
	$(CTR) image export artifacts/containerd-shim-process.tar $(CONTAINERD_SHIM_PROCESS_IMAGE)
	mkdir -p $(@D)
	cp artifacts/containerd-shim-process.tar $@
	$(CHOWN) -R $(shell id -u):$(shell id -g) rpmbuild

rpmbuild/SOURCES/docker.service: ../systemd/docker.service
	mkdir -p $(@D)
	cp $< $@

rpmbuild/SOURCES/dockerd.json: ../common/dockerd.json
	mkdir -p $(@D)
	sed \
	    -e 's!$${ENGINE_IMAGE}!$(ENGINE_IMAGE)!' \
	    -e 's!$${ENGINE_SCOPE}!$(ENGINE_SCOPE)!' \
	    -e 's/$${IMAGE_TAG}/$(IMAGE_TAG)/' \
	    $< > $@

# TODO: Eventually clean this up when we release an image with a manifest
DOCKER2OCI=artifacts/docker2oci
$(DOCKER2OCI):
	-$(CHOWN) -R $(shell id -u):$(shell id -g) $(@D)
	docker run --name docker2oci $(GO_IMAGE) sh -c 'go get github.com/coolljt0725/docker2oci'
	mkdir -p $(@D)
	docker cp docker2oci:/go/bin/docker2oci "$@"
	docker rm -f docker2oci
	$(CHOWN) -R $(shell id -u):$(shell id -g) $(@D)

# offline bundle
rpmbuild/SOURCES/engine.tar: $(DOCKER2OCI)
	$(MAKE) -C ../image ENGINE_IMAGE=$(ENGINE_IMAGE) ENGINE_SCOPE=$(ENGINE_SCOPE) image-linux
	mkdir -p artifacts
	docker save -o artifacts/docker-engine.tar $$(cat ../image/image-linux)
	./$(DOCKER2OCI) -i artifacts/docker-engine.tar artifacts/engine-image
	mkdir -p $(@D)
	tar c -C artifacts/engine-image . > $@
