# dockerfile git to push to gitops repo
FROM alpine/git:latest
RUN apk add --no-cache git curl

RUN curl --silent --fail --location \
    "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.8.1/kustomize_v5.8.1_linux_amd64.tar.gz" \
    | tar -xz -C /usr/local/bin && \
    chmod +x /usr/local/bin/kustomize

ADD entry.sh /entry.sh


RUN chmod +x /entry.sh

ENTRYPOINT [ "sh", "/entry.sh" ]