## Lab 16b - Securing Application access with OAuth (Bookinfo) <a name="Lab-16b"></a>
In this step, we're going to secure the access to the `httpbin` service using OAuth. Integrating an app with extauth consists of a few steps:
```
- create app registration in your OIDC
- configuring a Gloo Mesh `ExtAuthPolicy` and `ExtAuthServer`
- configuring the `RouteTable` with a specified label (i.e. `oauth: "true"`)
```

### In your OIDC Provider
Once the app has been configured in the external OIDC, we need to create a Kubernetes Secret that contains the OIDC client-secret. Please provide this value input before running the command below:
```bash
export BOOKINFO_CLIENT_SECRET="<provide OIDC client secret here>"
```

```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: bookinfo-oidc-client-secret
  namespace: bookinfo-frontends
type: extauth.solo.io/oauth
data:
  client-secret: $(echo -n ${BOOKINFO_CLIENT_SECRET} | base64)
EOF
```

Set the callback URL in your OIDC provider to map to our httpbin app
```bash
export APP_CALLBACK_URL="https://$(kubectl --context ${MGMT} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}')/productpage"

echo $APP_CALLBACK_URL
```

Lastly, replace the `OIDC_CLIENT_ID` and `ISSUER_URL` values below with your OIDC app settings:
```bash
export OIDC_CLIENT_ID="<client ID for httpbin app>"
export ISSUER_URL="<OIDC issuer url (i.e. https://dev-22651234.okta.com/oauth2/default)>"
```

Let's make sure our variables are set correctly:
```bash
echo $OIDC_CLIENT_ID
echo $ISSUER_URL
```

### Create ExtAuthPolicy
Now we will create an `ExtAuthPolicy`, which is a CRD that contains authentication information. 
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: bookinfo
  namespace: bookinfo-frontends
spec:
  applyToRoutes:
  - route:
      labels:
        oauth: "true"
  config:
    server:
      name: mgmt-ext-auth-server
      namespace: bookinfo-frontends
      cluster: mgmt
    glooAuth:
      configs:
      - oauth2:
          oidcAuthorizationCode:
            appUrl: ${APP_CALLBACK_URL}
            callbackPath: /oidc-callback
            clientId: ${OIDC_CLIENT_ID}
            clientSecretRef:
              name: bookinfo-oidc-client-secret
              namespace: bookinfo-frontends
            issuerUrl: ${ISSUER_URL}
            session:
              failOnFetchFailure: true
              redis:
                cookieName: bookinfo-session
                options:
                  host: redis.gloo-mesh-addons:6379
                allowRefreshing: false
              cookieOptions:
                maxAge: "1800"
            scopes:
            - email
            - profile
            logoutPath: /logout
            afterLogoutUrl: /productpage
            headers:
              idTokenHeader: Jwt
EOF
```

After that, you need to create an `ExtAuthServer`, which is a CRD that define which extauth server to use: 
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: ExtAuthServer
metadata:
  name: mgmt-ext-auth-server
  namespace: bookinfo-frontends
spec:
  destinationServer:
    ref:
      cluster: mgmt
      name: ext-auth-service
      namespace: gloo-mesh-addons
    port:
      name: grpc
EOF
```

Finally, you need to update the `RouteTable` to use this `AuthConfig`:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: productpage
  namespace: bookinfo-frontends
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: productpage
      labels:
        oauth: "true"
      matchers:
      - uri:
          exact: /productpage
      - uri:
          prefix: /static
      - uri:
          exact: /login
      - uri:
          exact: /logout
      - uri:
          prefix: /api/v1/products
      - uri:
          prefix: /oidc-callback
      forwardTo:
        destinations:
          - ref:
              name: productpage
              namespace: bookinfo-frontends
            port:
              number: 9080
EOF
```

## Cleanup

First let's apply the original bookinfo `RouteTable` yaml:
```bash
kubectl --context ${MGMT} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: productpage
  namespace: bookinfo-frontends
  labels:
    expose: "true"
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw-443
      namespace: istio-gateways
      cluster: mgmt
  workloadSelectors: []
  http:
    - name: productpage
      matchers:
      - uri:
          exact: /productpage
      - uri:
          prefix: /static
      - uri:
          exact: /login
      - uri:
          exact: /logout
      - uri:
          prefix: /api/v1/products
      forwardTo:
        destinations:
          - ref:
              name: productpage
              namespace: bookinfo-frontends
            port:
              number: 9080
EOF
```

Lets clean up the OAuth policies we've created:
```
# extauth config
kubectl --context ${MGMT} -n bookinfo-frontends delete ExtAuthPolicy bookinfo
kubectl --context ${MGMT} -n bookinfo-frontends delete secret bookinfo-oidc-client-secret
kubectl --context ${MGMT} -n bookinfo-frontends delete ExtAuthServer mgmt-ext-auth-server
```
