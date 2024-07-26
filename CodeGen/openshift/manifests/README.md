<h1 align="center" id="title">Deploy CodeGen on OpenShift Cluster</h1>

## Prerequisites

1. **Red Hat OpenShift Cluster** with dynamic *StorageClass* to provision *PersistentVolumes* e.g. **OpenShift Data Foundation**)
2. Exposed image registry to push there docker images (https://docs.openshift.com/container-platform/4.16/registry/securing-exposing-registry.html).
3. Access to https://huggingface.co/ to get access token with *Read* permissions. Update the access token in your repository:
```
cd GenAIExamples/CodeGen/openshift/manifests/xeon
export HUGGINGFACEHUB_API_TOKEN="YourOwnToken"
sed -i "s/insert-your-huggingface-token-here/${HUGGINGFACEHUB_API_TOKEN}/g" codegen.yaml
```
You can also customize the "MODEL_ID" and "model-volume".

## Deploy CodeGen On Xeon

1. Build docker images locally (https://github.com/opea-project/GenAIExamples/tree/main/CodeGen/docker/xeon#-build-docker-images)

- LLM Docker Image:
```
git clone https://github.com/opea-project/GenAIComps.git
cd GenAIComps
docker build -t opea/llm-tgi:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/llms/text-generation/tgi/Dockerfile .
```
- MegaService Docker Image:
```
git clone https://github.com/opea-project/GenAIExamples
cd GenAIExamples/CodeGen/docker
docker build -t opea/codegen:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f Dockerfile .
```
- UI Docker Image:
```
cd GenAIExamples/CodeGen/docker/ui/
docker build -t opea/codegen-ui:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f ./docker/Dockerfile .
```
To verify run the command: `docker images`.

2. Login to podman, tag the images and push it to image registry with following commands:

```
podman login -u <user> -p $(oc whoami -t) <openshift-image-registry_route> --tls-verify=false
podman tag <image_id> <openshift-image-registry_route>/<namespace>/<image_name>:<tag>
podman push <openshift-image-registry_route>/<namespace>/<image_name>:<tag>
```
To verify run the command: `oc get istag`.

3. Update images names in manifest files:

```
cd GenAIExamples/CodeGen/openshift/manifests/xeon
export IMAGE_LLM_TGI="YourImage"
export IMAGE_CODEGEN="YourImage"
export IMAGE_CODEGEN_UI="YourImage"
sed -i "s#insert-your-image-llm-tgi#${IMAGE_LLM_TGI}#g" codegen.yaml
sed -i "s#insert-your-image-codegen#${IMAGE_CODEGEN}#g" codegen.yaml
sed -i "s#insert-your-image-codegen-ui#${IMAGE_CODEGEN_UI}#g" ui-server.yaml
```

4. Deploy CodeGen with command:
```
oc apply -f codegen.yaml
```

5. Check the *codegen* route with command `oc get routes` and update the route in *ui-server.yaml* file: 
```
cd GenAIExamples/CodeGen/openshift/manifests/xeon
export CODEGEN_ROUTE="YourCodegenRoute"
sed -i "s/insert-your-codegen-route/${CODEGEN_ROUTE}/g" ui-server.yaml
```

6. Deploy UI with command:
```
oc apply -f ui-server.yaml
```

## Verify Services

Make sure all the pods are running, and restart the codegen-xxxx pod if necessary.

```
oc get pods
curl http://${CODEGEN_ROUTE}/v1/codegen -H "Content-Type: application/json" -d '{
     "messages": "Implement a high-level API for a TODO list application. The API takes as input an operation request and updates the TODO list in place. If the request is invalid, raise an exception."
     }'
```

## Launch the UI

To access the frontend, find the route for *ui-server* with command `oc get routes` and open it in the browser.
