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

Openshift can be used as a cloud provider using Kubernetes support plugin integrated in current versions of TeamCity. It allows to start agents as pods on demand and stop them then they are not needed.

Cloud Profile definition is managed on a TeamCity project level.

Profile parameters:
* name
* Cloud type: Kubernetes
* Server URL: TeamCity address (https://teamcity.mydomain.local)
* Kubernetes API server URL - Openshift API server address (https://api.ocp.mydomain.local:6443)
* Kubernetes namespace
* Authentication strategy: service account token for example
* Token: content of the service account token

Agent container images:
* name prefix - agent pod names will be prefixed with this value
* pod specification: custom pod template
* pod template - pod.yaml
* agent pool: project pool

When a job is defined in order for it to go to agents running as pods in Openshift in its Agent requirements specify a selector containing the name prefix. 

### Openshift namespace
Requirements:
* permissions to define cluster roles, service account, roles and security constraints definiion for service accounts
* a project / namespace is created
* a cluster role ocp-project-deployer is created (ocp-project-deployer.yaml)
* a service account is created in the project (serviceaccount.ayml)
* SCC allowing static UIDs in containers is created
* rolebinding between ocp-project-deployer role and service account
* context of SA token

Agents are started on demand. When not needed an agent pod is stopped.

JetBrains TeamCity agent container is built the way that it is not running with root user but rather with buildagent user, defined with static UID=1000. This practice is normal in Kubernetes. But Openshift defines more strict rules on UIDs requiring to run with random UIDs from a given range. In order for the container to start a cluster policy is to be created: `oc adm policy add-scc-to-user nonroot -z tc-serviceaccount --as system:admin`

### Self-signed certificates
TBD