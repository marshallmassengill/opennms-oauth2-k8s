## Setup Keycloak
 1. Create a new realm (or use an existing realm but the realm must be defined within Oauth2-Proxy as well.
 2. Define a new client that is enabled for the standard OIDC flow and has client authentication enabled.
 3. Define client scopes that are within the "dedicated" client scope: 
   - Scope 1:
     - Name: username
     - Mapper Type: User Attribute
     - User Attribute: username
     - Token Claim Name: preferred_username
     - Claim JSON Type: String
     - Add to ID Token: Enabled
     - Add to Access Token: Enabled
   - Scope 2:
     - Name: client-group-mapper
     - Mapper Type: Group Membership
     - Token Claim Name: clientGroupMapper
     - Add to ID Token: Enabled
     - Add to Access Token: Enabled
     - Add to userinfo: Enabled
   - Scope 3:
     - Name: aud-mapper-client
     - Mapper Type: Audience
     - Included Client Audience: oauth2-proxy (match it to your client name)
     - Add to ID Token: Disabled
     - Add to Access Token: Enabled
 4. Disable "Full scope allowed" for the dedicated client scopes.
 5. Define the groups that align with their names to the SECURITY ROLES for OpenNMS.  For a user this should be ROLE_USER.  For an administrator, that needs to be both ROLE_ADMIN,ROLE_USER.  More roles are listed here: https://docs.opennms.com/horizon/30/operation/user-management/security-roles.html
    - This is just one method for handling group membership and roles.  You can play with keycloak and the claim it is using for this and get other methods to work that will match your environment.

## Setup Oauth2-proxy
These are the key configuration items needed:
    	
    redirect_url = "https://onms-core.opennms.microk8s.dagonships.com/oauth2/callback"
    reverse_proxy = true
    set_xauthrequest = true
    pass_user_headers = true
    oidc_groups_claim="clientGroupMapper"

## Setup OpenNMS
Three files should be modified to enable OpenNMS to handle external auth.

 - $OPENNMS_HOME/etc/opennms.properties.d/jetty.properties
 - $OPENNMS_HOME/jetty-webapps/opennms/WEB-INF/spring-security.d/header-preauth.xml
 - $OPENNMS_HOME/jetty-webapps/opennms/WEB-INF/applicationContext-spring-security.xml

One will add pre-authorization and the other will add pre-authentication.  You need both for users who are not "entitled" within OpenNMS.

## Now Do it with K8s:
	
### Get the OpenNMS helm chart:
https://github.com/OpenNMS/helm-charts
### Stand up OpenNMS dependencies for the OpenNMS helm chart:
	scripts/start-dependencies.sh
### Setup your OpenNMS opennms-values.yaml file:
	{pull from github link}
### Create a new namespace:
	kubectl create namespace opennms
### Install the configmaps for OpenNMS (these don't have to be configmaps but probably should be:
	kubectl create configmap -n opennms jetty-properties --from-file=jetty.properties
	kubectl create configmap -n opennms header-preauth --from-file=header-preauth.xml
	kubectl create configmap -n opennms applicationcontext-spring-security --from-file=applicationContext-spring-security.xml
### Install the OpenNMS helm chart with values file:
	helm install opennms opennms/horizon -f opennms-values.yaml --set domain=microk8s.dagonships.com --namespace opennms
### Install the oauth2-proxy:
	helm install oauth2-proxy oauth2-proxy/oauth2-proxy -f oauth2-values.yaml --namespace opennms
### Additional Info:
#### OpenNMS pre-auth can be tested with:

    curl --insecure -H "x-auth-request-preferred-username: myuser" -H "x-auth-request-groups: ROLE_ADMIN,ROLE_USER,ROLE_REST" https://onms-core.opennms.microk8s.dagonships.com/opennms/rest/whoami

#### Amend the ingress for opennms:
	nginx.ingress.kubernetes.io/auth-response-headers: >-
          X-Auth-Request-Preferred-Username, X-Auth-Request-User,
          X-Auth-Request-Email, X-Auth-Request-Groups
	nginx.ingress.kubernetes.io/auth-signin: https://$host/oauth2/start?rd=$scheme://$host$request_uri
	nginx.ingress.kubernetes.io/auth-url: https://$host/oauth2/auth

Note: Out of the box, the OpenNMS helm chart uses the "release name" as a sub-domain.  This is to enable multiple deployments of OpenNMS and provide some means of differentiating them.  This can be modified in the ingress for the deployment but should be kept in mind.
