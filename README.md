# JetBrains TeamCity agent on Openshift4 

This project describes how to use vendor provided container of TeamCity agent on Openshift 4 clusters.

TeamCity supports integration with Kubernetes clusters where it can start on demand agents as pods. Openshift is not mentioned in the documentation.
Compared to Kubernetes Openshift provides a more restrictive permissions model, which has to be accounted for when starting an agent.

## Container

JetBrains provides an agent container at DockerHub: `jetbrains/teamcity-agent`.

## Local execution

It is possible to run an agent on a host with a container runtime (docker, podman, etc). 

The container requires an URL of a TeamCity server it is going to connect to: `SERVER_URL=https://teamcity.mydomain.local`

In case TeamCity runs with a self-signed certificate it has to be passed to a container so the agent would trus the server.

Get the certificate: 
```
openssl s_client -connect teamcity.mydomain.local:443 2>/dev/null </dev/null |  \
   sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > CERTIFICATE_NAME.pem
chmod 644 CERTIFICATE_NAME.pem
```

Start the agent container:
```
podman run -d --rm -e SERVER_URL=https://teamcity.mydomain.local \
  -v /tmp/CERTIFICATE_NAME.pem:/opt/buildagent/conf/trustedCertificates/teamcity.pem \
  jetbrains/teamcity-agent
```

Note: in order to be able to share artefacts with the container additional RW volumes can be mounted.

## Openshift

### TeamCity server

### Openshift namespace 