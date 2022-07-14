# buildah
```
sudo apt install qemu-user-static podman skopeo jq docker.io
sudo adduser $USER docker
sudo su - $USER

docker ps
podman ps

echo ok > index.html

cat <<'EOF'> Dockerfile
FROM docker.io/library/nginx:stable
COPY index.html /usr/share/nginx/html/index.html
RUN set -ex \
    && dpkg --print-architecture
EOF

# Run the lastest Buildah inside Podman
mkdir /tmp/containers
podman run -it --rm --privileged \
  -v $PWD:/app -w /app \
  -v /tmp/containers:/var/lib/containers \
  quay.io/buildah/stable bash

buildah images | awk '{print $3}' | xargs buildah rmi -f

buildah build --format=docker --jobs=2 \
  --platform=linux/arm64/v8,linux/amd64 \
  --manifest localhost:5000/website:v1 .

buildah manifest push --all \
  localhost:5000/website:v1 \
  oci-archive:website.tar

# Exit Podman
exit

skopeo inspect --raw oci-archive:website.tar | jq '.manifests[].platform.architecture'

podman run -d --rm --name registry -p 5000:5000 docker.io/library/registry:2

skopeo copy --all \
  oci-archive:website.tar \
  docker://localhost:5000/website:v1 \
  --dest-tls-verify=false

skopeo inspect --tls-verify=false --raw docker://localhost:5000/website:v1 | jq .

skopeo copy --all \
  oci-archive:website.tar \
  docker://localhost:5000/website:latest \
  --dest-tls-verify=false

skopeo copy --override-arch=amd64 \
  docker://localhost:5000/website:v1 \
  docker://localhost:5000/website:v1-amd64 \
  --dest-tls-verify=false \
  --src-tls-verify=false

skopeo copy --override-arch=arm64 \
  docker://localhost:5000/website:v1 \
  docker://localhost:5000/website:v1-arm64 \
  --dest-tls-verify=false \
  --src-tls-verify=false

skopeo inspect --tls-verify=false docker://localhost:5000/website:v1 | jq .

podman run -it --rm --tls-verify=false -p 8080:80 localhost:5000/website:v1

curl http://localhost:8080

skopeo copy --override-arch=amd64 \
  oci-archive:website.tar \
  docker-archive:website-amd64.tar

skopeo inspect docker-archive:website-amd64.tar | jq

skopeo copy --override-arch=arm64 \
  oci-archive:website.tar \
  docker-archive:website-arm64.tar

skopeo inspect docker-archive:website-arm64.tar | jq

docker load -i website-amd64.tar | \
  awk -F':' '{print $NF}' | \
  xargs -i docker tag {} website-amd64

docker run -it --rm website-amd64

docker load -i website-arm64.tar | \
  awk -F':' '{print $NF}' | \
  xargs -i docker tag {} website-arm64

docker run -it --rm website-arm64
```
