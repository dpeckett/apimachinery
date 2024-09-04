VERSION 0.8
FROM golang:1.22-bookworm
WORKDIR /workspace

all:
  COPY +package/*.deb /workspace/dist/
  RUN cd dist && find . -type f | sort | xargs sha256sum >> ../sha256sums.txt
  SAVE ARTIFACT ./dist/*.deb AS LOCAL dist/
  SAVE ARTIFACT ./sha256sums.txt AS LOCAL dist/sha256sums.txt

tidy:
  LOCALLY
  ENV GOTOOLCHAIN=go1.22.1
  RUN go mod tidy
  RUN go fmt ./...

lint:
  FROM golangci/golangci-lint:v1.59.1
  WORKDIR /workspace
  COPY . .
  RUN golangci-lint run --timeout 5m ./...

test:
  COPY go.mod go.sum .
  RUN go mod download
  COPY . .
  RUN go test -coverprofile=coverage.out -v ./...
  SAVE ARTIFACT coverage.out AS LOCAL coverage.out

package:
  FROM debian:bookworm
  # Use bookworm-backports for newer golang versions
  RUN echo "deb http://deb.debian.org/debian bookworm-backports main" > /etc/apt/sources.list.d/backports.list
  RUN apt update
  # Tooling
  RUN apt install -y git devscripts dpkg-dev debhelper-compat dh-sequence-golang \
    golang-any=2:1.22~3~bpo12+1 golang-go=2:1.22~3~bpo12+1 golang-src=2:1.22~3~bpo12+1
  # Build Dependencies
  RUN apt install -y \
    golang-github-davecgh-go-spew-dev \
    golang-github-elazarl-goproxy-dev \
    golang-github-evanphx-json-patch-dev \
    golang-github-gogo-protobuf-dev \
    golang-github-golang-groupcache-dev \
    golang-github-golang-protobuf-1-3-dev \
    golang-github-google-go-cmp-dev \
    golang-github-google-gofuzz-dev \
    golang-github-google-uuid-dev \
    golang-github-googleapis-gnostic-dev \
    golang-github-hashicorp-golang-lru-dev \
    golang-github-json-iterator-go-dev \
    golang-github-docker-spdystream-dev \
    golang-github-modern-go-reflect2-dev \
    golang-github-mxk-go-flowrate-dev \
    golang-github-spf13-pflag-dev \
    golang-github-stretchr-testify-dev \
    golang-golang-x-net-dev \
    golang-gopkg-inf.v0-dev \
    golang-gopkg-yaml.v2-dev \
    golang-k8s-klog-dev \
    golang-k8s-kube-openapi-dev \
    golang-k8s-sigs-structured-merge-diff-dev \
    golang-k8s-sigs-yaml-dev
  RUN mkdir -p /workspace/golang-k8s-apimachinery
  WORKDIR /workspace/golang-k8s-apimachinery
  COPY . .
  # Fix a compatibility issue with debians gnostic package
  RUN sed -i 's|github.com/googleapis/gnostic/openapiv2|github.com/googleapis/gnostic/OpenAPIv2|g' \
    pkg/util/strategicpatch/testing/openapi.go
  ENV EMAIL=damian@pecke.tt
  RUN export DEBEMAIL="damian@pecke.tt" \
    && export DEBFULLNAME="Damian Peckett" \
    && export VERSION="0.29.0" \
    && export DEBVERSION="0.0~git$(git log -1 --date=format:%Y%m%d --pretty=format:%cd).$(git rev-parse --short HEAD)" \
    && dch --create --package golang-k8s-apimachinery --newversion "${VERSION}-${DEBVERSION}" \
      --distribution "UNRELEASED" --force-distribution  --controlmaint "Last Commit: $(git log -1 --pretty=format:'(%ai) %H %cn <%ce>')" \
    && tar -czf ../golang-k8s-apimachinery_${VERSION}.orig.tar.gz .
  RUN dpkg-buildpackage -us -uc
  SAVE ARTIFACT /workspace/*.deb AS LOCAL dist/