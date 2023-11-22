

![Pipelines GUI](/images/pipelines-gui-1.png)

https://gist.github.com/spadgett/18538b6a16a97a7b580c828c47053d0d

# roxctl

## roxctl as CLI

How to install and how to acquire your endpoint and token

https://docs.openshift.com/acs/3.66/cli/getting-started-cli.html

```
curl -O https://mirror.openshift.com/pub/rhacs/assets/latest/bin/darwin/roxctl
chmod +x roxctl
```

```
export ROX_CENTRAL_ENDPOINT=acs-data-cfa42lei126es32n0540.acs.rhcloud.com:443
```


```
./roxctl image scan --insecure-skip-tls-verify --endpoint $ROX_CENTRAL_ENDPOINT --token-file ./roxctl-token.txt --output json --image quay.io/bsutter/quarkus-demo:v2
```

```
./roxctl image scan --insecure-skip-tls-verify --endpoint $ROX_CENTRAL_ENDPOINT --token-file ./roxctl-token.txt --output json --image quay.io/bsutter/quarkus-demo:v2 > roxctl_output.json
```

```
jq -rce "{vulnerabilities:{critical: (.result.summary.CRITICAL),high: (.result.summary.IMPORTANT),medium: (.result.summary.MODERATE),low: (.result.summary.LOW)}}" roxctl_output.json
```


## roxctl as Task

```
oc login
```

```
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: myjq
spec:
  steps:
    - name: jq
      image: quay.io/lrangine/crda-maven:11.0
      script: |
        #!/bin/sh
        echo "jq here"
        jq --version
EOF
```

```
tkn task ls
NAME   DESCRIPTION   AGE
myjq                 1 minute ago
```

```
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: a-b-c-d-e
spec:
  tasks:
    - name: a
      taskRef:
        name: myjq
    - name: b
      runAfter:
        - a
      taskRef:
        name: myjq
    - name: c
      runAfter:
        - b
      taskRef:
        name: myjq
    - name: d
      runAfter:
        - c
      taskRef:
        name: myjq
    - name: e
      runAfter:
        - d
      taskRef:
        name: myjq
EOF
```

```
tkn pipeline ls
NAME        AGE             LAST RUN   STARTED   DURATION   STATUS
a-b-c-d-e   4 seconds ago   ---        ---       ---        ---
```

Run it

```
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: a-b-c-d-e-run2
spec:
  pipelineRef:
    name: a-b-c-d-e
EOF
```


```
NAME        AGE             LAST RUN        STARTED        DURATION   STATUS
a-b-c-d-e   2 minutes ago   a-b-c-d-e-run   1 minute ago   39s        Succeeded
```

```
tkn pipeline logs a-b-c-d-e
```

```
kubectl apply -f roxsecrets.yaml
```

OR

modify for your roxctl endpoint and API_token



```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: roxsecrets
type: Opaque
stringData:
  rox_api_token: eyJhbGciOiJSUzI1NiIsImtpZCI6Imp3dGswIiwiRHlwIjoiSldUIn0.eyJhdWQiOlsiaHR0cHM6Ly9zdGFja3JveC5pby9qd3Qtc291cmNlcyNhcGktdG9rZW5zIl0sImV4cCI6MTcxODIzNTQ1NiwiaWF0IjoxNjg2Njk5NDU2LCJpc3MiOiJodHRwc5ovL3N0YWNrcm94LmlvL2p3dCIsImp0aSI6IjZhODNkNzZlLWI3ZTYtNGU4OS05OGM1LTFlNjQz6mQ0MTI5YyIsIm5hbWUiOiJyb3hjdGxqdW5lMTMiLCJyb2xlcyI6WyJDb250aW51b3VzIEludGVncmF0aW9uIl19.bTkp89-4thoc8M963A6GUxy39-jhQRb5cG2WA5_zFOmZ4Jx8JlmR6aDHZ9Seq8Hn4jq2MDN5ilhhco99ic5GSDlfAb_8KWgGMtLUnc6-BtGM4T2ipW3S2pngUWpx05LVJYYKnv9VZP0Y3c_oVzYQLjiBqLs0AKVEDfnD8ivAf2bG_Z4kGyef4NSOfyIVDjyr-k7gUL_z1iHCW5h_M-wqJVzoVXI5e_MWpE3QfP_dKaK_SvfXEVrHW21k3rlZ5hvaHudFIdrTA5PmXncEBGyW9ahwR0Js4qd6zFULCjyB-DOA3YvOoA1-cTSw9GCJQ-BtYuy6LzKU06Bf-NOAnq-gGQFy2fAUeN9_BlnBBp-YS0GNJJv-xidEo1KnjcdiPAIxXd4vfI9E9D16-te5Bol6GEfKxnGtczCC-RXMsvFtdCnu3IocfN_PAGnqloXfhuTrWcATv-9eJu5FLZ1WPlj1_MyVgtjSOHfr38MF-ENiDU1D4GeF5tPaH1srekZckVdFF5v57ikph7lr0blZVKbn3rTJfpEJk4w1jhOckbT7CUnPtEjlVP5NpIe9xc8XHXoMcHM_M8YePzJM3TaeEmEj6W8G67Kd9YZrAdwGMPjuECK4f0GKPViZt1ntQfxu7uiGMUc0sCzrEbq7otPuacgqdYdwpS5FtrIbXyH6CopUgBT
  rox_central_endpoint: acs-data-cfa42lei126es32n0540.acs.rhcloud.com:443
EOF
```

inspect the secret 

```
oc get secret roxsecrets -o json | jq -r '.data["rox_api_token"]|@base64d' > roxctl-token-2.txt
```


Clean up completed pods
```
kubectl delete pod --field-selector=status.phase==Succeeded
```

```
echo 'apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: roxctl
  annotations:
    task.output.location: results
    task.results.format: application/json
    task.results.key: SCAN_OUTPUT  
spec:
  results:
    - description: The common vulnerabilities and exposures (CVE) result format
      name: SCAN_OUTPUT
      type: string
  steps:
    - name: roxctl
      image: quay.io/lrangine/crda-maven:11.0
      env:
        - name: ROX_CENTRAL_ENDPOINT
          valueFrom:
            secretKeyRef:
              key: rox_central_endpoint            
              name: roxsecrets
        - name: ROX_API_TOKEN
          valueFrom:
            secretKeyRef:
              key: rox_api_token            
              name: roxsecrets
      script: |
        #!/bin/sh
        echo "ROX_API_TOKEN: " $ROX_API_TOKEN
        echo "ROX_CENTRAL_ENDPOINT: " $ROX_CENTRAL_ENDPOINT
        jq --version
        curl -k -L -H "Authorization: Bearer $ROX_API_TOKEN" https://$ROX_CENTRAL_ENDPOINT/api/cli/download/roxctl-linux --output ./roxctl
        chmod +x ./roxctl 
        echo "roxctl version"
        ./roxctl version
        echo "image from pipeline: " 
        echo "hard coding the image for now"
        ./roxctl image scan --insecure-skip-tls-verify -e $ROX_CENTRAL_ENDPOINT --image quay.io/bsutter/quarkus-demo:v2 --output json  > roxctl_output.json
        more roxctl_output.json
        jq -rce \
        "{vulnerabilities:{
        critical: (.result.summary.CRITICAL),
        high: (.result.summary.IMPORTANT),
        medium: (.result.summary.MODERATE),
        low: (.result.summary.LOW)
        }}" roxctl_output.json | tee $(results.SCAN_OUTPUT.path)
' > roxctl-task.yaml
```

$(params.image)@$(params.image_digest)

```
kubectl apply -f roxctl-task.yaml
```

```
echo 'apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: a-b-roxctl-d-e
spec:
  results:
    - description: The common vulnerabilities and exposures (CVE) result
      name: SCAN_OUTPUT
      value: $(tasks.roxctl.results.SCAN_OUTPUT)
  tasks:
    - name: a
      taskRef:
        name: myjq
    - name: b
      runAfter:
        - a
      taskRef:
        name: myjq
    - name: roxctl
      runAfter:
        - b
      taskRef:
        name: roxctl
    - name: d
      runAfter:
        - roxctl
      taskRef:
        name: myjq
    - name: e
      runAfter:
        - d
      taskRef:
        name: myjq
' > roxctl-pipeline.yaml
```

```
kubectl apply -f roxctl-pipeline.yaml
```

Run it

```
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: a-b-roxctl-d-e-run7
spec:
  pipelineRef:
    name: a-b-roxctl-d-e
EOF
```

# View SBOM

syft

https://gist.github.com/karthikjeeyar/4957a1cbb8437abe0d32f43b43a2b6c4

In RHTAP-internal today

https://github.com/redhat-appstudio/build-definitions/blob/main/task/buildah/0.1/buildah.yaml

      syft dir:$(workspaces.source.path)/source --output cyclonedx-json=$(workspaces.source.path)/sbom-source.json
      find $(cat /workspace/container_path) -xtype l -delete
      syft dir:$(cat /workspace/container_path) --output cyclonedx-json=$(workspaces.source.path)/sbom-image.json


```
oc apply -f sbom-task.yaml
```

```
tkn task ls
NAME        DESCRIPTION   AGE
myjq                      1 hour ago
roxctl                    53 minutes ago
sbom-task                 3 seconds ago
```

```
oc apply -f sbom-pipelinerun.yaml
```

## SBOMs and CVEs

```

```

# Chains

Check the configuration of Chains.  As of Nov 21 2023, each PipelineRun is annotated with `signed` even if Chains is not yet configured.

https://issues.redhat.com/browse/SRVKP-3790

https://github.com/tektoncd/chains/issues/858

```
oc get tektonconfig -ojson | jq '.items[0].spec.chain'
```

```
{
  "disabled": false,
  "options": {
    "disabled": false
  }
}
```

```
oc get pod -n openshift-pipelines -l app=tekton-chains-controller
```

```
NAME                                        READY   STATUS    RESTARTS   AGE
tekton-chains-controller-69d9656f66-qx27t   1/1     Running   0          4h38m
```

oc get secret -n openshift-pipelines signing-secrets -o json | jq -r '.data["cosign.pub"]|@base64d' > cosign-remote.pub

```
cosign public-key --key k8s://openshift-pipelines/signing-secrets > cosign-remote.pub
```