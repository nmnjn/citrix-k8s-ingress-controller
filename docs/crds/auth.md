# Define authentication and authorization policies on the Ingress Citrix ADC

Authentication and authorization policies are used to enforce access restrictions to the resources hosted by an application or API server. While you can verify the identity using the authentication policies, authorization policies are used to verify whether a specified request has the necessary permissions to access a resource.

Citrix provides a Kubernetes [CustomResourceDefinition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) (CRD) called the **Auth CRD** that you can use with the Citrix ingress controller to define authentication policies on the ingress Citrix ADC.

## Auth CRD definition

The Auth CRD is available in the Citrix ingress controller GitHub repo at: [auth-crd.yaml](https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/crd/auth/auth-crd.yaml). The Auth CRD provides [attributes](#auth-crd-attributes) for the various options that are required to define the authentication policies on the Ingress Citrix ADC.

The following is the Auth CRD definition:

```yml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: authpolicies.citrix.com
spec:
  group: citrix.com
  version: v1beta1
  names:
    kind: authpolicy
    plural: authpolicies
    singular: authpolicy 
  scope: Namespaced
  subresources:
    status: {}
  additionalPrinterColumns:
    - name: Status
      type: string
      description: 'Current Status of the CRD'
      JSONPath: .status.state
    - name: Message
      type: string
      description: 'Status Message'
      JSONPath: .status.status_message
  validation:
    openAPIV3Schema:
      type: object 
      properties:
        spec:
          type: object 
          properties:
            servicenames:
              description: |+
                           'Name of the services for which the policies applied'
              type: array
              items:
                type: string
                maxLength: 63
            authentication_mechanism:
              type: object 
              description: |+
                          'Authentication mechanism. Options: using forms or using request header.
                           Default is Authentication using request header, when no option is specified'
              properties:
                    using_request_header:
                      description: |+
                                   'Enable user authentication using request header. Use when the credentials
                                    or api keys are passed in a header. For example, when using Basic, Digest,
                                    Bearer authentication or api keys.
                                    When authentication using forms is provided, this is set to OFF'

                      type: string
                    using_forms:
                      type: object
                      description: 'Enables authentication using forms. Use with user/web authentication.'
                      properties:
                        authentication_host:
                          description: |+
                                       'Fully qualified domain name (FQDN) for authentication
                                        FQDN to which the user must be redirected for
                                        authentication. This FQDN should be unique and should resolve to frontend IP of
                                        ADC with Ingress/service type LoadBalancer'
                          type: string
                          maxLength: 255
                        authentication_host_cert:
                          description: |+
                                       'Name of the SSL certificate to be used with authentication_host.
                                        This certificate is mandatory while using_forms'
                          type: object
                          properties:
                              tls_secret:
                                type: string
                                description: 'Name of the Kubernetes Secret of type tls referring to Certificate'
                                pattern: '^[a-z0-9]([-a-z0-9]*[a-z0-9])?$'
                              preconfigured:
                                type: string
                                maxLength: 63
                                description: |+
                                             'Preconfigured SSL certkey name on ADC with the
                                              certificate and key already added on ADC'
                          oneOf:
                          - properties:
                            required: [tls_secret]
                          - properties:             
                            required: [preconfigured]
                        ingress_name:
                          description: |+
                                       'Ingress name for which the authentication using forms
                                        is applicable.'
                          type: string
                          maxLength: 63
                        lb_service_name:
                          description: |+
                                       'Service of type LoadBalancer for which the authentication using forms
                                        is applicable.'
                          type: string
                          maxLength: 63
                        vip:
                          description: |+
                                       'Frontend IP of ingress for which the authentication 
                                        using forms is applicable. This refers to frontend-ip provided 
                                        with Ingress. It is suggested to use vip, if more than one Ingress
                                        resource use the same frontend-ip'
                          type: string
                      required: [authentication_host, authentication_host_cert]
                      oneOf:
                      - properties:
                        required: [ingress_name]                      
                      - properties:
                        required: [lb_service_name]
                      - properties:
                        required: [vip]
              oneOf:
                - properties:
                    using_request_header:
                      enum: ['ON']
                  required: [using_request_header]
                - properties:
                  required: [using_forms]
                  
            authentication_providers:
              description: |+
                           'Authentication Configuration for required authentication providers/schemes.
                            One or more of these can be created'
              type: array
              items:
                  description: 'Create config for a single authentication provider of a particular type'
                  type: object 
                  properties:
                    name:
                      description: 'Name for this provider, has to be unique, referenced by authentication policies'
                      type: string
                      maxLength: 127

                    oauth:
                      description: 'Authentication provided by external oAuth provider' 
                      type: object 
                      properties:
                          issuer:
                              description: 'Identity of the server whose tokens are to be accepted'
                              type: string
                              maxLength: 127
                          audience:
                              description: 'Audience for which token sent by Authorization server is applicable'
                              type: array
                              items:
                                type: string
                                maxLength: 127
                          jwks_uri:
                              description: |+
                                          'URL of the endpoint that contains JWKs (Json Web Key) for 
                                           JWT (Json Web Token) verification'
                              type: string
                              maxLength: 127
                          introspect_url:
                              description: ' URL of the introspection server'
                              type: string
                              maxLength: 127
                          client_credentials:
                              description: |+
                                           'secrets object that contains Client Id and secret as known 
                                            to Introspection server'
                              type: string
                              maxLength: 253
                          token_in_hdr:
                              description: |+
                                           'custom header name where token is present,
                                            default is Authorization header'
                              type: array
                              items:
                                type: string
                                maxLength: 127
                              maxItems: 2
                          token_in_param:
                              description: 'query parameter name where token is present'
                              type: array
                              items:
                                type: string
                                maxLength: 127
                              maxItems: 2
                          signature_algorithms:
                              description: 'list of allowed signature algorithms, by default HS256, RS256, RS512 are allowed'
                              type: array
                              items:
                                type: string
                                enum: ['HS256', 'RS256', 'RS512']
                          claims_to_save:
                              description: 'list of claims to be saved, used to create authorization policies'
                              type: array
                              items:
                                type: string
                                maxLength: 127
                          metadata_url:
                              description: 'URL used to get OAUTH/OIDC provider metadata'
                              type: string
                              maxLength: 255
                          user_field:
                              description: |+
                                           'Attribute in the token from which username should be extracted. 
                                            by default, ADC looks at email attribute for user id'
                              type: string
                              maxLength: 127
                          default_group:
                              description: |+
                                           'group assigned to the request if authentication succeeds,
                                            this is in addition to any extracted groups from token'
                              type: string
                              maxLength: 63
                          grant_type:
                              description: 'used to specify the type of flow to the token end point, defaults to CODE'
                              type: array
                              items:
                                type: string
                                enum: ['CODE','PASSWORD']
                          pkce:
                              description: 'specify whether to enable Proof Key Code Exchange, defaults to ENABLED'
                              type: string
                              enum: ['ENABLED', 'DISABLED']
                          token_ep_auth_method:
                              description: |+
                                           'authentication method to be used with token end point, 
                                            defaults to client_secret_post'
                              type: string
                              enum: ['client_secret_post', 'client_secret_jwt']

                      anyOf:
                      - properties:
                        required : [jwks_uri]
                      - properties:
                        required : [introspect_url, client_credentials]
                      - properties:
                        required : [metadata_url]

                    ldap:
                      description: 'LDAP authentication provider'
                      type: object
                      properties:
                          server_ip:
                              description: 'IP address assigned to the LDAP server'
                              type: string
                          server_name:
                              description: 'LDAP server name as a FQDN'
                              type: string
                              maxLength: 127
                          server_port:
                              description: 'Port on which the LDAP server accepts connections. Default is 389'
                              type: integer
                              minimum: 1
                              maximum: 65535
                          base:
                              description: |+
                                           'Base (node) from which to start LDAP searches. If the LDAP server is
                                            running locally, the default value of base is dc=netscaler, dc=com'
                              type: string
                              maxLength: 127
                          server_login_credentials:
                              description: |+
                                           'Kubernetes secret object providing credentials to login to LDAP server,
                                            The secret data should have username and password'
                              type: string
                          login_name:
                              description: |+
                                           'LDAP login name attribute. The Citrix ADC uses the LDAP login name
                                            to query external LDAP servers or Active Directories'
                              type: string
                              maxLength: 127
                          security_type:
                              description: |+
                                           'Type of security used for communications between the Citrix ADC
                                            and the LDAP server. Default is TLS'
                              type: string
                              enum: ['PLAINTEXT', 'TLS', 'SSL']
                          validate_server_cert:
                              description: 'Validate LDAP Server certs. Default is NO'
                              type: string
                              enum: ['YES', 'NO']
                          hostname:
                              description: |+
                                           'Hostname for the LDAP server. If validate_server_cert is ON,
                                            this must be the host name on the certificate from the LDAP
                                            A hostname mismatch will cause a connection failure'
                              type: string 
                              maxLength: 127
                          sub_attribute_name:
                              description: 'LDAP group sub-attribute name. Used for group extraction from the LDAP server.'
                              type: string
                              maxLength: 31
                          group_attribute_name:
                              description: 'LDAP group attribute name. Used for group extraction on the LDAP server.'
                              type: string
                              maxLength: 31
                          search_filter:
                              description: |+
                                           'String to be combined with the default LDAP user search string to form the
                                            search value. For example, if the search filter "vpnallowed=true" is combined
                                            with the LDAP login name "samaccount" and the user-supplied username is "bob",
                                            the result is the LDAP search string ""(&(vpnallowed=true)(samaccount=bob)"" 
                                            (Be sure to enclose the search string in two sets of double quotation marks)'
                              type: string
                              maxLength: 255
                          auth_timeout:
                              description: |+
                                           'Number of seconds the Citrix ADC waits for a response from the server
                                            Default is 3'
                              type: integer
                              minimum: 1
                              maximum: 4294967295
                          password_change:
                              description: 'Allow password change requests. Default is DISABLED'
                              type: string
                              enum: ['ENABLED', 'DISABLED']
                          attributes_to_save:
                              description: |+
                                           'List of attribute names separated by comma which needs to be fetched
                                            from LDAP server and stored as key-value pair for the session on ADC'
                              type: string
                              maxLength: 2047
                      oneOf:
                        - properties:
                          required: [server_ip]
                        - properties:
                          required: [server_name]
                          
                    saml:
                      description: |+
                                   'SAML authentication provider.
                                    Currently SAML is supported only with authentication mechanism using forms'
                      type: object
                      properties:
                          metadata_url:
                              description: 'URL is used for obtaining saml metadata.'
                              type: string
                              maxLength: 255
                          metadata_refresh_interval:
                              description: |+
                                           'Interval in minutes for fetching metadata from specified metadata URL.
                                            Default is 36000'
                              type: integer
                              minimum: 1
                              maximum: 4294967295
                          signing_cert:
                              description: 'SSL certificate to sign requests from SP to IDP'
                              type: object
                              properties:
                                  tls_secret:
                                    type: string
                                    description: 'Name of the Kubernetes Secret of type tls referring to Certificate'
                                    pattern: '^[a-z0-9]([-a-z0-9]*[a-z0-9])?$'
                                  preconfigured:
                                    type: string
                                    maxLength: 63
                                    description: |+
                                                 'Preconfigured SSL certkey name on ADC with the
                                                  certificate and key already added on ADC'                                    
                              oneOf:
                              - properties:
                                required: [tls_secret]
                              - properties:
                                required: [preconfigured]
                          audience:
                              description: 'Audience for which assertion sent by IdP is applicable'
                              type: string
                              maxLength: 127                          
                          issuer_name:
                              description: 'The name to be used in requests sent from SP to IDP to identify citrix ADC'
                              type: string
                              maxLength: 63
                          binding:
                              description: 'Specifies the transport mechanism of saml message. Default is POST'
                              type: string
                              enum: ['REDIRECT', 'POST', 'ARTIFACT']
                          artifact_resolution_service_url:
                              description: 'URL of the Artifact Resolution Service on IdP'
                              type: string
                              maxLength: 255
                          logout_binding:
                              description: 'Specifies the transport mechanism of saml logout.  Default is POST'
                              type: string
                              enum: ['REDIRECT', 'POST']
                          reject_unsigned_assertion:
                              description: |+
                                           'Reject unsigned SAML assertions. ON, rejects assertion without signature.
                                            STRICT ensure that both Response and Assertion are signed. Default is ON'
                              type: string
                              enum: ['ON', 'OFF', 'STRICT']                      
                          user_field:
                              description: 'SAML user ID, as given in the SAML assertion'
                              type: string
                              maxLength: 63
                          default_authentication_group:
                              description: |+
                                           'This is the default group that is chosen when the authentication 
                                            succeeds in addition to extracted groups'
                              type: string
                              maxLength: 63                                                  
                          skew_time:
                              description: |+
                                           'Allowed clock skew in number of minutes on an incoming assertion.
                                            Default is 5'
                              type: integer
                              minimum: 1
                          attributes_to_save:
                              description: |+
                                           'List of attribute names separated by comma which needs to be extracted
                                            and stored as key-value pair for the session on ADC'
                              type: string
                              maxLength: 2047                   
                      required:
                        - metadata_url

                    basic_local_db:
                      description: 'Basic HTTP authentication, user data in local DB of ADC'

                  required:
                    - name

            authentication_policies:
              description: 'Authentication policies'
              type: array
              items:
                type: object 
                description: 'Authentication policy'
                properties:
                  resource:
                      type: object 
                      description: 'endpoint/resource selection criteria'
                      properties:
                        path:
                          description: 'api resource path e.g. /products. '
                          type: array
                          items:
                            type: string
                            maxLength: 511
                        method:
                          type: array
                          items:
                            type: string
                            enum: ['GET', 'PUT', 'POST','DELETE']
                      required:
                        - path
                  provider:
                    description: 'name of the authentication provider for the policy, empty if no authentication required'
                    type: array
                    items:
                      type: string
                      maxLength: 127
                    maxItems: 1
                required:
                  - resource
                  - provider

            authorization_policies:
              description: 'Authorization policies'
              type: array
              items:
                type: object 
                description: 'Authorization policy'
                properties:
                  resource:
                      type: object 
                      description: 'endpoint/resource selection criteria'
                      properties:
                        path:
                          description: 'api resource path e.g. /products. '
                          type: array
                          items:
                            type: string
                            maxLength: 511
                        method:
                          description: ' http method'
                          type: array
                          items:
                            type: string
                            enum: ['GET', 'PUT', 'POST','DELETE']
                        claims:
                          description: 'authorization scopes required for selected resource saved as claims or attributes'
                          type: array
                          items:
                              type: object
                              properties:
                                name:
                                  description: 'name of the claim/attribute to check'
                                  type: string
                                  maxLength: 127
                                values:
                                  description: 'list of claim values required for the request'
                                  type: array
                                  items:
                                    type: string
                                    maxLength: 127
                                  minItems: 1
                              required:
                                - name
                                - values
                      required:
                        - claims

          required:
            - servicenames      
```

## Auth CRD attributes

The Auth CRD provides the following attributes that you use to define the authentication policies:

-  `servicenames`
-  `authentication_mechanism`
-  `authentication_providers`
-  `authentication_policies`
-  `authorization_policies`
  
### Servicenames

The name of the services that you want to bind to the authentication policy.

### Authentication mechanism

The following authentication mechanisms are supported:

- Using request headers:  
Enables user authentication using the request header. You can use this mechanism when the credentials or API keys are passed in a header. For example, you can use authentication using request headers for basic, digest, bearer authentication, or API keys.

- Using forms:
Enables authentication using forms. You can use this mechanism with user or web authentication.

When the authentication mechanism is not specified, the default is authentication using the request header.

The following are the attributes for forms based authentication.

| Attribute | Description |
| --------- | ----------- |
| `authentication_host` | Specifies a fully qualified domain name (FQDN) to which the user must be redirected for authentication. This FQDN should be unique and should resolve to the front-end IP address of Citrix ADC with Ingress or service type LoadBalancer.|
| `authentication_host_cert` | Specifies the name of the SSL certificate to be used with the `authentication_host`. This certificate is mandatory while performing authentication using the form.|
|`ingress_name`| Specifies the Ingress name for which the authentication using forms is applicable.|
| `lb_service_name`| Specifies the name of the service of type LoadBalancer for which the authentication using forms is applicable.|
| `vip` |Specifies the front-end IP address of the Ingress for which the authentication using forms is applicable. This attribute refers to the `frontend-ip` address provided with the Ingress. If there is more than one Ingress resource which uses the same frontend-ip, it is recommended to use vip.|

**Note:** While using forms, authentication can be enabled for all types of traffic. Currently, granular authentication is not supported.

### Authentication providers

The **providers** define the authentication mechanism and parameters that are required for the authentication mechanism. The CRD supports both *basic authentication* and *OAuth authentication*.

#### Basic authentication

Specifies that local authentication is used with the HTTP basic authentication scheme. To use basic authentication, you must create user accounts on the ingress Citrix ADC.

#### OAuth authentication

The OAuth authentication mechanism, requires an external identity provider to authenticate the client using oAuth2 and issue an Access token. When the client presents the Access token to a Citrix ADC as an access credential, the Citrix ADC validates the token using the configured values. If the token validation is successful then Citrix ADC grants access to the client.

##### OAuth authentication attributes

The following are the attributes for OAuth authentication:

| Attribute | Description |
| --------- | ----------- |
| `Issuer` | The identity (usually a URL) of the server whose tokens need to be accepted for authentication.|
| `jwks_uri` | The URL of the endpoint that contains JWKs (JSON Web Key) for JWT (JSON Web Token) verification.|
| `audience` | The identity of the service or application for which the token is applicable.|
| `token_in_hdr` | The custom header name where the token is present. The default value is the `Authorization` header.</br> **Note:** You can specify more than one header.|
| `token_in_param` | The query parameter where the token is present.|
|`signature_algorithms`| Specifies the list of signature algorithms which are allowed. By default HS256, RS256, and RS512 algorithms are allowed.|
| `introspect_url`| The URL of the introspection endpoint of the authentication server (IdP). If the access token presented is an opaque token, introspection is used for the token verification.|
| `client_credentials`| The name of the Kubernetes secrets object that contains the client id and client secret required to authenticate with the authentication server.|
| `claims_to_save`| The list of claims to be saved. Claims are used to create authorization policies.|

OpenID Connect (OIDC) is a simple identity layer on top of the OAuth 2.0 protocol. OIDC allows clients to verify the identity of the end-user based on the authentication performed by an authorization server, as well as to obtain basic profile information about the end-user. In addition to the OAuth attributes, you can use the following attributes to configure OIDC.

| Attribute | Description |
| --------- | ----------- |
| `metadata_url` | Specifies the URL that is used to get OAUTH or OIDC provider metadata.|
| `user_field` | Specifies the attribute in the token from which the user name should be extracted. By default, Citrix ADC examines the email attribute for user ID.|
| `default_group` | Specifies the group assigned to the request if authentication succeeds. This group is in addition to any extracted groups from the token. |
| `grant_type` | Specifies the type of flow to the token end point. The default value is `CODE`.|
| `pkce` | Specifies whether to enable Proof Key for Code Exchange (PKCE). The default value is `ENABLED`.|
| `token_ep_auth_method` | Specifies the authentication method to be used with the token end point. The default value is `client_secret_post`.|

#### SAML authentication

Security assertion markup language (SAML) is an XML-based open standard which enables authentication of users across products or organizations. The SAML authentication mechanism, requires an external identity provider to authenticate the client. SAML works by transferring the client identity from the identity provider to the Citrix ADC. On successful validation of the client identity, the Citrix ADC grants access to the client.

The following are the attributes for SAML authentication.

| Attribute | Description |
| --------- | ----------- |
| `metadata_url` | Specifies the URL used for obtaining SAML metadata. |
| `metadata_refresh_interval` | Specifies the interval in minutes for fetching metadata from the specified metadata URL.|
| `signing_cert` | Specifies the SSL certificate to sign requests from the service provider (SP) to the identity provider (IdP).|
| `audience` | Specifies the identity of the service or application for which the token is applicable.|
| `issuer_name` | Specifies the name used in requests sent from SP to IdP to identify the Citrix ADC.|
| `binding` | Specifies the transport mechanism of the SAML message. The default value is `POST`.|
| `artifact_resolution_service_url` | Specifies the URL of the artifact resolution service on IdP.|
| `logout_binding` | Specifies the transport mechanism of the SAML logout. The default value is `POST`.|
|`reject_unsigned_assertion`| Rejects unsigned SAML assertions. If this value is `ON`, it rejects assertion without signature.|
|`user_field`| Specifies the SAML user ID specified in the SAML assertion|
|`default_authentication_group`| Specifies the default group that is chosen when the authentication succeeds in addition to extracted groups.|
|`skewtime`| Specifies the allowed clock skew time in minutes on an incoming SAML assertion.|
|`attributes_to_save`| Specifies the list of attribute names separated by commas which needs to be extracted and stored as key-value pairs for the session on Citrix ADC.|

#### LDAP authentication

LDAP (Lightweight Directory Access Protocol) is an open, vendor-neutral, industry standard application protocol for accessing and maintaining distributed directory information services over an Internet Protocol (IP) network. A common use of LDAP is to provide a central place to store user names and passwords. LDAP allows many different applications and services to connect to the LDAP server to validate users.

**Note:** LDAP authentication is supported through both the authentication mechanisms using the request header or using forms.

The following are the attributes for LDAP authentication.

| Attribute | Description |
| --------- | ----------- |
| `server_ip` | Specifies the IP address assigned to the LDAP server. |
| `server_name` | Specifies the LDAP server name as an FQDN.|
| `server_port` | Specifies the port on which the LDAP server accepts connections. The default value is 389.|
| `base` | Specifies the base node on which to start LDAP searches. If the LDAP server is running locally, the default value of base is `dc=netscaler`, `dc=com`.|
| `server_login_credentials` | Specifies the Kubernetes secret object providing credentials to log in to the LDAP server. The secret data should have user name and password.|
| `login_name` | Specifies the **LDAP login name** attribute. The Citrix ADC uses the LDAP login name to query external LDAP servers or Active Directories.|
| `security_type` | Specifies the type of security used for communications between the Citrix ADC and the LDAP server. The default is TLS.|
| `validate_server_cert` | Validates LDAP server certificates. The default value is `NO`.|
|`hostname`|Specifies the host name for the LDAP server. If `validate_server_cert` is `ON`, this value must be the host name on the certificate from the LDAP. A host name mismatch causes a connection failure.|
|`sub_attribute_name`| Specifies the LDAP group subattribute name. This attribute is used for group extraction from the LDAP server.|
|`group_attribute_name`| Specifies the LDAP group attribute name. This attribute is used for group extraction on the LDAP server.|
|`search_filter`| Specifies the string to be combined with the default LDAP user search string to form the search value. For example, if the search filter "vpnallowed=true" is combined with the LDAP login name "samaccount" and the user-supplied user name is "bob", the result is the LDAP search string ""(&(vpnallowed=true)(samaccount=bob)"". Enclose the search string in two sets of double quotation marks.|
|`auth_timeout`| Specifies the number of seconds the Citrix ADC waits for a response from the server. The default value is 3.|
|`password_change`|Allows password change requests. The default value is `DISABLED`. |
|`attributes_to_save`| List of attribute names separated by comma which needs to be fetched from the LDAP server and stored as key-value pairs for the session on Citrix ADC. |

### Authentication policies

The **authentication_policies** allow you to define the traffic selection criteria to apply the authentication mechanism and also to specify the provider that you want to use for the selected traffic.

The following are the attributes for policies:

| Attribute | Description |
| --------- | ----------- |
| `path` | An array of URL path prefixes that refer to a specific API endpoint. For example, `/api/v1/products/`.  |
| `method` | An array of HTTP methods. Allowed values are GET, PUT, POST, or DELETE. </br>**Note:** The traffic is selected if the incoming request URI matches with any of the paths AND any of the listed methods. If the method is not specified then the path alone is used for the traffic selection criteria.|
| `provider` | Specifies the authentication mechanism that needs to be used. If the authentication mechanism is not provided, then authentication is not performed.|

### Authorization policies

Authorization policies allow you to define the traffic selection criteria to apply the authorization requirements for the selected traffic.

The following are the attributes for authorization policies:

| Attribute | Description |
| --------- | ----------- |
| `path` | An array of URL path prefixes that refer to a specific API endpoint. For example, `/api/v1/products/`.  |
| `method` | An array of HTTP methods. Allowed values are GET, PUT, POST, or DELETE. </br>**Note:** The traffic is selected if the incoming request URI matches with any of the paths AND any of the listed methods. If the method is not specified then the path alone is used for the traffic selection criteria.|
| `claims` | Specifies the claims required to access a specific API endpoint. `name` indicates the claim name and `values` indicate the required permissions. You can claim more than one claim. If an empty list is specified, it implies that authorization is not required. |

## Deploy the Auth CRD

Perform the following to deploy the Auth CRD:

1.  Download the CRD ([auth-crd.yaml](https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/crd/auth/auth-crd.yaml)).

2.  Deploy the Auth CRD using the following command:

        kubectl create -f auth-crd.yaml

    For example:

        root@master:~# kubectl create -f auth-crd.yaml

        customresourcedefinition.apiextensions.k8s.io/authpolicies.citrix.com created

## How to write the authentication policies

After you have deployed the CRD provided by Citrix in the Kubernetes cluster, you can define the authentication policy configuration in a `.yaml` file. In the `.yaml` file, use `authpolicy` in the `kind` field and in the `spec` section add the **Auth CRD** attributes based on your requirement for the policy configuration.

After you deploy the `.yaml` file, the Citrix ingress controller applies the authentication policy configuration on the Ingress Citrix ADC device.

The following is a sample authentication policy definition ([auth_example1.yaml](https://raw.githubusercontent.com/citrix/citrix-k8s-ingress-controller/master/crd/auth/auth_example1.yaml)):

```yml
apiVersion: citrix.com/v1beta1
kind: authpolicy
metadata:
  name: authexample
spec:
    servicenames:
    - frontend

    authentication_providers:

        - name: "local-auth-provider"
          basic-local-db:

        - name: "jwt-auth-provider"
          oauth:
            issuer: "https://sts.windows.net/tenant1/"
            jwks_uri: "https://login.microsoftonline.com/tenant1/discovery/v2.0/keys"
            audience : ["https://vault.azure.net"]
            claims_to_save : ["scope"]
        
        - name: "introspect-provider"
          oauth:
            issuer: "ns-idp"
            jwks_uri: "https://idp.aaa/oauth/idp/certs”
            audience : ["https://api.service.net"]
            client_credentials: "oauthsecret"
            introspect_url: https://idp.aaa/oauth/idp/introspect
            claims_to_save : ["scope"]

    authentication_policies:

        - resource:
            path:
              - '/orders/'
              - '/shipping/'
            method: [GET, POST]
          provider: ["local-auth-provider"]

        - resource:
            path:
              - '/products/'
            method: [POST]
          provider: ["local-auth-provider"]

          # no auth for this
        - resource:
            path:
              - '/products/'
            method: [GET]
          provider: []

          # oauth provider for this
        - resource:
            path:
              -  '/reviews/'
          provider: ["jwt-auth-provider"]

          # introspection provider for this
        - resource:
            path:
              -  '/customers/'
          provider: ["introspect-provider"]
    
    authorization_policies:

        - resource:
            path:
              - '/customers/'
            method: [POST]
            claims: 
             - name: "scope"
               values: ["read", "write"]

        - resource:
            path:
              - '/reviews'
            claims: 
             - name: "scope"
               values: ["read"]
        - resource:
            path:
              - '/products/'
            method: [GET]
            claims: []


```

The sample authentication policy performs the following:

-  The Citrix ADC performs the authentication mechanism specified in the provider `local-auth-provider` on the requests to the following endpoints:
      -  **orders**, **shipping**, and **GET** or **POST**
      -  **products** and **POST**
  
-  The Citrix ADC does not perform the authentication for the **products** and **GET** endpoints.

-  The Citrix ADC performs the OAuth JWT verification as specified in the provider `jwt-auth-provider` for the requests to the **reviews** endpoint.

-  The Citrix ADC performs the OAuth introspection as specified in the provider `introspect-provider` for the requests to the **customers** endpoint.
  
-  The Citrix ADC requires the `scope` claim with `read` and `write` permissions to access the **customers** endpoint and **POST**.
  
-  The Citrix ADC does not need any authorization permissions to access the **products** endpoint with GET operation.

For OAuth, if the token is present in a custom header, it can be specified using the `token_in_hdr` attribute as follows:


          oauth:
            issuer: "https://sts.windows.net/tenant1/"
            jwks_uri: "https://login.microsoftonline.com/tenant1/discovery/v2.0/keys"
            audience : ["https://vault.azure.net"]
            token_in_hdr : [“custom-hdr1”]

Similarly, if the token is present in a query parameter, it can be specified using the `token_in_param` attribute as follows:

          oauth:
            issuer: "https://sts.windows.net/tenant1/"
            jwks_uri: "https://login.microsoftonline.com/tenant1/discovery/v2.0keys"
            audience : ["https://vault.azure.net"]
            token_in_param : [“query-param1”]

### Creating a secrets object with client credentials for introspection

A Kubernetes secrets object is needed for configuring the OAuth introspection.
You can create a secret object in a similar way as shown in the following example:


    apiVersion: v1        
    kind: Secret          
    metadata:             
      name: oauthsecret
    type: Opaque        
    stringData:           
     client_id: "nsintro"
     client_secret: "nssintro"

**Note:**
Keys of the opaque secret object must be `client_id` and `client_secret`. A user can set the values for them as desired.

### SAML authentication using forms

The following is an example for SAML authentication using forms. In the example, `authhost-tls-cert-secret` and `saml-tls-cert-secret` are Kubernetes TLS secrets referring to certificate and key.

**Note:** When `certkey.cert` and `certkey.key` are certificate and key respectively for the authentication host, then the `authhost-tls-cert-secret` can be formed using the following command:

         kubectl create secret tls authhost-tls-cert-secret --key="certkey.key" --cert="certkey.cert

  Similarly, you can use this command to form `saml-tls-cert-secret` with the required certificate and key.

```yml

apiVersion: citrix.com/v1beta1
kind: authpolicy
metadata:
  name: samlexample
spec:
    servicenames:
    - frontend

    authentication_mechanism:
      using_forms:
        authentication_host: "fqdn_authenticaton_host"
        authentication_host_cert:
          tls_secret: authhost-tls-cert-secret
        vip: "192.2.156.156"

    authentication_providers:
        - name: "saml-auth-provider"
          saml:
              metadata_url: "https://idp.aaa/metadata/samlidp/aaa"
              signing_cert:
                  tls_secret: saml-tls-cert-secret

    authentication_policies:

        - resource:
            path: []
            method: []
          provider: ["saml-auth-provider"]

    authorization_policies:

        - resource:
            path: []
            method: []
            claims: []

```

### OpenID Connect authentication using forms

The following is an example for creating OpenID Connect authentication to configure Citrix ADC in a Relaying Party (RP) role to authenticate users for an external identity provider. The `authentication_mechanism` must be set to `using_forms` to trigger the OpenID Connect procedures.

```yml
apiVersion: citrix.com/v1beta1
kind: authpolicy
metadata:
  name: authoidc
spec:
    servicenames:
    - frontend
    authentication_mechanism:
        using_forms:
            authentication_host: "10.221.35.213"
            authentication_host_cert:
                 tls_secret: "oidc-tls-secret"
            vip: "10.221.35.213"

    authentication_providers:

        - name: "oidc-provider"
          oauth:
            audience : ["https://app1.citrix.com"]
            client_credentials: "oidcsecret"
            metadata_url: "https://10.221.35.214/oauth/idp/.well-known/openid-configuration"
            default_group: "groupA"
            user_field: "sub"
            pkce: "ENABLED"
            token_ep_auth_method: "client_secret_post"

    authentication_policies:

        - resource:
            path: []
            method: []
          provider: ["oidc-provider"]

    authorization_policies:

        #default - no authorization requirements
        - resource:
            path: []
            method: []
            claims: []
```

### LDAP authentication using the request header

The following is an example for LDAP authentication using the request header.

In this example, `ldapcredential` is the Kubernetes secret referring to the LDAP server credentials. See the `ldap_secret.yaml` file for information on how to create LDAP server credentials.

```yml

apiVersion: citrix.com/v1beta1
kind: authpolicy
metadata:
  name: ldapexample
spec:
    servicenames:
    - frontend

    authentication_providers:
        - name: "ldap-auth-provider"
          ldap:
              server_ip: "192.2.156.160"
              base: 'dc=aaa,dc=local'
              login_name: accountname
              sub_attribute_name: CN
              server_login_credentials: ldapcredential

        - name: "local-auth-provider"
          basic-local-db:

    authentication_policies:

        - resource:
            path: []
            method: []
          provider: ["ldap-auth-provider"]


    authorization_policies:

        - resource:
            path: []
            method: []
            claims: []
```

**Note:** With the request header based authentication mechanism, granular authentication based on traffic is supported.

### LDAP authentication using forms

In the example `authhost-tls-cert-secret` is the Kubernetes TLS secret referring to certificate and key.

When `certkey.cert` and `certkey.key` are certificate and key respectively for the authentication host, then the `authhost-tls-cert-secret`  can be formed using the following
 command:

        kubectl create secret tls authhost-tls-cert-secret --key="certkey.key" --cert="certkey.cert

In this example, `ldapcredential` is the Kubernetes secret referring to the LDAP server credentials. See the `ldap_secret.yaml` file for information on how to create LDAP server credentials.

```yml
apiVersion: citrix.com/v1beta1
kind: authpolicy
metadata:
  name: ldapexample
spec:
    servicenames:
    - frontend

    authentication_mechanism:
      using_forms:
        authentication_host: "fqdn_authenticaton_host"
        authentication_host_cert:
          tls_secret: authhost-tls-cert-secret
        vip: "192.2.156.156"

    authentication_providers:
        - name: "ldap-auth-provider"
          ldap:
              server_ip: "192.2.156.160"
              base: 'dc=aaa,dc=local'
              login_name: accountname
              sub_attribute_name: CN
              server_login_credentials: ldapcredential

    authentication_policies:

        - resource:
            path: []
            method: []
          provider: ["ldap-auth-provider"]

    authorization_policies:

        - resource:
            path: []
            method: []
            claims: []

```

**LDAP_secret.yaml**

The following is an example for `LDAP_secret.yaml`.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ldapcredential
type: Opaque
stringData:
  username: 'ldap_server_username'
  password: 'ldap_server_password'

```
        