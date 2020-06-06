# ocp-operator-catalog-pipeline

start artifactory container:
```
podman run --name artifactory -d -v /opt/jfrog/artifactory:/var/opt/jfrog/artifactory --ulimit nofile=90000:90000 -p 8081:8081 docker.bintray.io/jfrog/artifactory-pro:6.16.2
```

some environment varialbles:
```
ARTIFACTORY_URL=10.0.0.14:8081
```

to push files to the artifactory:
```
curl -uadmin:AP75wUAsRv7ZX8nS8zDCa4Xbz5v -T redhat-operators.tar.gz "http://${ARTIFACTORY_URL}/artifactory/ocp-catalog/redhat-operators-5-27-20.tar.gz"
```

patch to add insecure registry if needed:
```
oc patch --type=merge --patch='{
"spec": {
    "registrySources": {
      "insecureRegistries": [
      "${ARTIFACTORY_URL}"
      ]
    }
  }
}' image.config.openshift.io/cluster
```
and do this on the crc vm:

```
$ crc ip
192.168.64.92

$ ssh -i ~/.crc/machines/crc/id_rsa -o StrictHostKeyChecking=no core@192.168.64.92

<CRC-VM> $ sudo cat /etc/containers/registries.conf 
unqualified-search-registries = ['registry.access.redhat.com', 'docker.io']

[[registry]]
  location = "my.insecure.registry.com:8888"
  insecure = true
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

<CRC-VM> $ sudo systemctl restart crio
<CRC-VM> $ sudo systemctl restart kubelet
<CRC-VM> $ exit
```

To disable the default catalog sources:
```
oc patch OperatorHub cluster --type json     -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
```

Make sure the pull secret is in place for artifactory:
```
oc create secret docker-registry artifactory-secret --docker-server=${ARTIFACTORY_URL} --docker-username=admin --docker-password=password --docker-email=lshulman@redhat.com
oc secrets link default artifactory-secret --for=pull
```


Ex:
```
oc process -f operator-registry-template.yaml -p NAME=citi -p EAR_OPERATOR_MANIFESTS_URL="http://${ARTIFACTORY_URL}/artifactory/ocp-catalog/ocp-catalog-1.0.2.tar.gz" -p CURL_EAR_CREDS="-uadmin:password" -p IMAGE=${ARTIFACTORY_URL}/docker-local/citi-ose-operator-registry:v4.3 | oc create -f -
```

