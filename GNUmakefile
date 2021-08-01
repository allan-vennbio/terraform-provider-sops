export CGO_ENABLED = 0
VERSION = $(shell git describe --tags --match='v*' --always)
RELEASE = $(patsubst v%,%,$(VERSION))# Remove leading v to comply with Terraform Registry conventions

CROSSBUILD_OS   = linux#windows darwin
CROSSBUILD_ARCH = amd64# arm64 386
SKIP_OSARCH     = darwin_386 windows_arm64
OSARCH_COMBOS   = $(filter-out $(SKIP_OSARCH),$(foreach os,$(CROSSBUILD_OS),$(addprefix $(os)_,$(CROSSBUILD_ARCH))))
RELEASE_FOLDER  = ~/.terraform.d/plugins/hashicorp.com/lokkersp/sops/$(VERSION)/linux_amd64


default: build

style:
	@echo ">> checking code style"
	! gofmt -d $(shell find . -name '*.go' -print) | grep '^'

vet:
	@echo ">> vetting code"
	go vet ./...

test:
	@echo ">> testing code"
	go test -v ./...

build:
	@echo ">> building binaries"
	go build -o terraform-provider-sops

crossbuild: $(GOPATH)/bin/gox
	@echo ">> cross-building"
	gox -arch="$(CROSSBUILD_ARCH)" -os="$(CROSSBUILD_OS)" -osarch="$(addprefix !,$(subst _,/,$(SKIP_OSARCH)))" \
		-output="binaries/$(VERSION)/{{.OS}}_{{.Arch}}/terraform-provider-sops_$(VERSION)"

$(GOPATH)/bin/gox:
	# Need to disable modules for this to not pollute go.mod
	@GO111MODULE=off go get -u github.com/mitchellh/gox

install: crossbuild
	@echo ">> install locally"
	mkdir -p $(RELEASE_FOLDER)
	mkdir -p $(RELEASE_SHARE_FOLDER)
	cp binaries/$(VERSION)/linux_amd64/terraform-provider-sops_$(VERSION) $(RELEASE_FOLDER)/terraform-provider-sops_$(VERSION)

release: crossbuild bin/hub
	@echo ">> uploading release $(VERSION)"
	mkdir -p releases
	set -e; for OSARCH in $(OSARCH_COMBOS); do \
		zip -j releases/terraform-provider-sops_$(RELEASE)_$$OSARCH.zip binaries/$(VERSION)/$$OSARCH/terraform-provider-sops_* > /dev/null; \
		./bin/hub release edit -m "" -a "releases/terraform-provider-sops_$(RELEASE)_$$OSARCH.zip#terraform-provider-sops_$(RELEASE)_$$OSARCH.zip" $(VERSION); \
	done
	@echo ">>> generating sha256sums:"
	cd releases; sha256sum *.zip | tee terraform-provider-sops_$(RELEASE)_SHA256SUMS
	./bin/hub release edit -m "" -a "releases/terraform-provider-sops_$(RELEASE)_SHA256SUMS#terraform-provider-sops_$(RELEASE)_SHA256SUMS" $(VERSION)

bin/hub:
	@mkdir -p bin
	curl -sL 'https://github.com/github/hub/releases/download/v2.14.1/hub-linux-amd64-2.14.1.tgz' | \
		tar -xzf - --strip-components 2 -C bin --wildcards '*/bin/hub'

.PHONY: all style vet test build crossbuild release
