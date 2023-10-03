# How this repo works to demonstrate API curation

From a TAP testbed, follow the following steps to set up the environment and experiment with API curation.

To learn more about the application itself, see [APPLICATION.md](APPLICATION.md).

## Operator actions

1. Get shared ingress domain

```bash
export INGRESS_DOMAIN="$(kubectl get secret tap-tap-install-values -n tap-install -o json | jq -r '.data."values.yaml"' | base64 --decode | yq .shared.ingress_domain)" 
```

1. Get TAP version

```bash
export TAP_VERSION="$(tanzu package available list tap.tanzu.vmware.com -n tap-install -ojson | jq -r .[0].version)"
```

1. (Optional) Create SSO AuthServer 

If you'd like to skip this step, please remove `ssoEnabled` from [your CuratedAPIDescriptor definition](./config/curated-profile-api.yaml) 
and remove the `ClientRegistration` resource from [gateway.yaml](./config/gateway.yaml).
If you'd like to test with SSO enabled on your SCG instances, you may set up an AuthServer with the following steps. 

```bash

sed -i".bak" "s/tap.test-api-curation.tapdemo.vmware.com/$INGRESS_DOMAIN/g" config/sso-config.yaml
kubectl apply -f config/sso-config.yaml

# wait for k8s resource to get ready
timeout --kill-after=30s 30s bash -c "while ! kubectl get -n appsso deployment/my-auth-server-auth-server; do sleep 5; done"
kubectl wait --for=condition=available -n appsso deployment/my-auth-server-auth-server --timeout=60s
kubectl wait --for=condition=available -n appsso deployment/my-auth-server-redis --timeout=60s
```

1. Install SCG and create SCG resource

```bash
tanzu package install spring-cloud-gateway -n tap-install -p spring-cloud-gateway.tanzu.vmware.com -v 2.1.0

kubectl create namespace gateway
sed -i".bak" "s/tap.test-api-curation.tapdemo.vmware.com/$INGRESS_DOMAIN/g" config/gateway.yaml
kubectl apply -f config/gateway.yaml
```

1. Update API Auto Registration with SCG is enabled and TAP GUI guest login enabled

```bash
mkdir tmp | true
kubectl get secret tap-tap-install-values -n tap-install -o json | jq -r '.data."values.yaml"' | base64 --decode > tmp/tap-values.yaml 
yq e '.tap_gui.app_config.auth.allowGuestAccess = true' -i tmp/tap-values.yaml
yq e '.api_auto_registration.route_provider.spring_cloud_gateway.enabled = true' -i tmp/tap-values.yaml
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tmp/tap-values.yaml -n tap-install
```

## Application developer actions

1. Create postgresql service claim for the app to run:

```bash
tanzu service class-claim create customer-database --class postgresql-unmanaged
tanzu service class-claim create employee-database --class postgresql-unmanaged
```

1. Create workload and make sure they are ready

```bash
tanzu apps workload apply -f config/workload-customer.yaml
tanzu apps workload apply -f config/workload-employee.yaml
```

## API Curator action

1. Create CuratedAPIDescriptor

```bash
kubectl apply -f config/curated-profile-api.yaml

kubectl get curatedapidescriptor -A # Should see the aggregated API spec URL if the resource is ready
```

## Verify curation working

### (Optional) With API portal

Update API portal with the sourceURL that would include all the curated API specs

```bash
yq e '.api_portal.apiPortalServer.sourceUrls = "http://api-auto-registration.tap.test-api-curation.tapdemo.vmware.com/openapi"' -i tmp/tap-values.yaml
sed -i".bak" "s/tap.test-api-curation.tapdemo.vmware.com/$INGRESS_DOMAIN/g" tmp/tap-values.yaml

tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tmp/tap-values.yaml -n tap-install
```

You may add individual spec URLs for specific curated APIs you like to API portal source URL locations.

Visit `http://api-portal.$INGRESS_DOMAIN` to see the newly curated API.

### API calls to the curated API directly

```bash
curl -XPOST http://profile-management.$INGRESS_DOMAIN/employees/profiles/ -H 'Content-Type: application/json' -d '{
  "email": "a@b.c",
  "firstName": "Sponge",
  "lastName": "Bob"
}'
curl http://profile-management.$INGRESS_DOMAIN/employees/profiles/ 
```

### Verify SCG filters

Visit `http://profile-management.$INGRESS_DOMAIN/customers/profiles` in browser and log in as `alice` with password `test`.

Refresh the page multiple times and should get `429` from the rate limit filter.

### Check for spec update

Commit some update about the endpoint and wait for supply chain to deploy the new app. Then the updated spec should be picked up by the `APIDescriptor` then the `CuratedAPIDescriptor` eventually. 
The resync time is default to be 5 minutes for each resource, but it can be configured by setting `api_auto_registration.sync_period` in values file to something shorter, e.g. 30s.  

## Clean up the cluster

```bash
kubectl delete namespace curation
kubectl delete namespace gateway
kubectl delete -f config/sso-config.yaml
tanzu apps workload delete -f config/workload-customer.yaml -y
tanzu apps workload delete -f config/workload-employee.yaml -y
tanzu service class-claim delete customer-database -y
tanzu service class-claim delete employee-database -y
```
