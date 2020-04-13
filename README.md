# Service Binding Specification

Specification for binding services to runtime applications running in Kubernetes.  

## Terminology definition

*  **service** - any software that is exposing functionality.  Could be a RESTful application, a database, an event stream, etc.
*  **application** - in this specification we refer to a single runtime-based microservice (e.g. MicroProfile app, or Node Express app) as an application.  This is different than an umbrella (SIG) _Application_ which refers to a set of microservices.
*  **binding** - providing the necessary information for an application to connect to a service.
*  **Secret** - refers to a Kubernetes [Secret](https://kubernetes.io/docs/concepts/configuration/secret/).
*  **ConfigMap** - refers to a Kubernetes [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).

## Motivation

*  Need a consistent way to bind k8s application to services (applications, databases, event streams, etc)
*  A standard / spec / RFC will enable adoption from different service providers
*  Cloud Foundry has addressed this issue with a [buildpack specification](https://github.com/buildpacks/rfcs/blob/master/text/0012-service-binding.md). The equivalent is not available for k8s.

## Proposal

Main section of the doc.  Has sub-sections that outline the design.

### Making a service bindable

#### Minimum requirements for being bindable
A bindable service **MUST** comply with one-of:
* provide a Secret and/or ConfigMap that contains the [binding data](#service-binding-schema) and reference this Secret and/or ConfigMap using one of the patterns discussed [below](#pointer-to-binding-data). 
* map its `status`, `spec`, `data` properties to the corresponding [binding data](#service-binding-schema), using one of the patterns discussed [below](#pointer-to-binding-data).
* include a sample `ServiceBinding` (see the [Request service binding](#Request-service-binding) section below) in its documentation (e.g. GitHub repository, installation instructions, etc) which contains a `dataMapping` illustrating how each of its `status` properties map to the corresponding [binding data](#service-binding-schema).  This option allows existing services to be bindable with zero code changes.

#### Recommended requirements for being bindable
In addition to the minimum set above, a bindable service **SHOULD** provide:
* a ConfigMap (which could be the same as the one holding some of the binding data, if applicable) that describes metadata associated with each of the items referenced in the Secret.  The bindable service should also provide a reference to this ConfigMap using one of the patterns discussed [below](#pointer-to-binding-data).

The key/value pairs insides this ConfigMap are:
* A set of `metadata.<property>=<value>` - where `<property>` maps to one of the defined keys for this service, and `<value>` represents the description of the value.  For example, this is useful to define what format the `password` key is in, such as apiKey, basic auth password, token, etc.


#### Pointer to binding data

This specification supports different scenarios for exposing bindable data. Below is a summary of how to indicate what is interesting for binding.  Please see the [annotation section](annotations.md) for the full set with more details.

1. OLM-enabled Operator: Use the `statusDescriptor` and/or `specDescriptor` parts of the CSV to mark which `status` and/or `spec` properties reference the [binding data](#service-binding-schema):
    * The reference's `x-descriptors` with a possible combination of:
      * ConfigMap:
        
            - path: data.dbcredentials
              x-descriptors:
                - urn:alm:descriptor:io.kubernetes:ConfigMap 
                - servicebinding

      * Secret:

            - path: data.dbcredentials
              x-descriptors:
                - urn:alm:descriptor:io.kubernetes:Secret 
                - servicebinding        

      * Individual binding items from a `Secret`:
        * ```
            - urn:alm:descriptor:io.kubernetes:Secret 
            - servicebinding:username
          ```
        * ```
            - urn:alm:descriptor:io.kubernetes:Secret 
            - servicebinding:password
          ```
      * Individual binding items from a `ConfigMap`:
        * ```
            - urn:alm:descriptor:io.kubernetes:ConfigMap 
            - servicebinding:port
          ```
        * ```
            - urn:alm:descriptor:io.kubernetes:ConfigMap
            - servicebinding:host
          ```
      * Individual backing items from a path referencing a string value
       
        * ```
            - path: data.uri
              x-descriptors:
                - servicebinding 
          ```

2. Non-OLM Operator: - An annotation in the Operator's CRD to mark which `status` and/or `spec` properties reference the [binding data](#service-binding-schema) :
      * ConfigMap:
        *   ```
             "servicebinding.dev/certificate":
             "path={.status.data.dbConfiguration},objectType=ConfigMap"
            ```

      * Secret:
        *   ```
             "servicebinding.dev/dbCredentials":
             "path={.status.data.dbCredentials},objectType=Secret"
            ```

      * Individual binding items from a `ConfigMap`
        *   ```
            “servicebinding.dev/host": 
            “path={.status.data.dbConfiguration},objectType=ConfigMap,sourceKey=address"
            ```
     
        *   ```
            “servicebinding.dev/port": 
            “path={.status.data.dbConfiguration},objectType=ConfigMap
            ```

      * Individual backing items from a path referencing a string value

        * ```
            “servicebinding.dev/uri”:"path={.status.data.connectionURL}"
          ```

      
3. Regular k8s resources (Ingress, Route, Service, Secret, ConfigMap etc)  - An annotation in the corresponding Kubernetes resources that maps the `status`, `spec` or `data` properties to their corresponding [binding data](#service-binding-schema). 

All annotations used in CRDs in the above section can be used for regular k8s resources, as well.

The above pattern can be used to expose external services (such as from a VM or external cluster), as long as there is an entity such as a Secret that provides the binding details. 

### Service Binding Schema

The core set of binding data is:
* **type** - the type of the service. Examples: openapi, db2, kafka, etc.
* **host** - the host (IP or host name) where the service resides.
* **port** - the port to access the service.
* **endpoints** - the endpoint information to access the service in a service-specific syntax, such as a list of hosts and ports. This is an alternative to separate `host` and `port` properties.
* **protocol** - the protocol of the service.  Examples: http, https, postgresql, mysql, mongodb, amqp, mqtt, kafka, etc.
* **basePath** - a URL prefix for this service, relative to the host root. It MUST start with a leading slash `/`.  Example: the URL prefix for a RESTful service. 
* **username** - the username to log into the service.  Can be omitted if no authorization required, or if equivalent information is provided in the password as a token.
* **password** - the password or token used to log into the service.  Can be omitted if no authorization required, or take another format such as an API key.  It is strongly recommended that the corresponding ConfigMap metadata properly describes this key.
* **certificate** - the certificate used by the client to connect to the service.  Can be omitted if no certificate is required, or simply point to another Secret that holds the client certificate.  
* **uri** - for convenience, the full URI of the service in the form of `<protocol>://<host>:<port>[<basePath>]`.
* **roleNeeded** - the name of the role needed to fetch the Secret containing the binding data.  In this scenario, a k8s Service Account with the appropriate role must be passed into the binding request (see the [RBAC](#rbac) section below).

Extra binding properties **can** also be defined (preferably with corresponding ConfigMap metadata) by the bindable service, using one of the patterns defined in [Pointer to binding data](#pointer-to-binding-data).

The data that is injected or mounted into the container may have a different name because of a few reasons:
* the backing service may have chosen a different name (discouraged, but allowed).
* a custom name may have been chosen using the `dataMappings` portion of the `ServiceBinding` CR.
* a prefix may have been added to certain items in `ServiceBinding`, via the usage of the `id` attribute for a specific backing service, or via the global `envVarPrefix` flag.

Application should rely on the `SERVICE_BINDINGS` environment variable for the accurate list of injected or mounted binding items, as [defined below](#Mounting-and-injecting-binding-information).


### Request service binding

Binding is requested by the consuming application, or an entity on its behalf such as the [Runtime Component Operator](https://github.com/application-stacks/runtime-component-operator), via a custom resource that is applied in the same cluster where an implementation of this specification resides.

Since the reference implementation for most of this specification is the [Service Binding Operator](https://github.com/redhat-developer/service-binding-operator) we will be using the `ServiceBinding` CRD, which resides in [this folder](https://github.com/redhat-developer/service-binding-operator/tree/master/deploy/crds), as the entity that holds the binding request.  

**Temporary Note**
To ensure a better fit with the specification a few modifications have been proposed to the `ServiceBinding` CRD:
* A modification to its API group, to be independent from OpenShift. ([ref](https://github.com/redhat-developer/service-binding-operator/issues/364))
* A simplification of its CRD name to `ServiceBinding` (we are already using this name in the spec). ([ref](https://github.com/redhat-developer/service-binding-operator/issues/365))
* Renaming `customEnvVar` to `dataMapping`. ([ref](https://github.com/redhat-developer/service-binding-operator/issues/356#issuecomment-595943295))
* Allowing for the `application` selector to be omitted, for the cases where another Operator owns the deployment. ([ref](https://github.com/redhat-developer/service-binding-operator/issues/296))
* Addition of fields such as `serviceAccount` and `subscriptionSecret` that support more advanced binding cases. ([ref](https://github.com/redhat-developer/service-binding-operator/issues/355))
* Not related to the CRD, but directly related to how this spec approaches item 1 from [Pointer to binding data](#pointer-to-binding-data), in terms of referencing a nested property.  ([ref](https://github.com/redhat-developer/service-binding-operator/issues/361))

#### Security

This part of the specification deals with security in three aspects:
1. does the user have the authority to create a ServiceBinding CR?
1. does the user have the authority to access the binding data for all requested services?
1. does the user have the authority to modify the source application with the injected binding data?

Scenario 1 can enforced by requiring a certain role type to create ServiceBinding CR, using k8s native RBAC rules.

Scenario 2 there are a few options, one of them being if the service provider's binding resource (as defined in [Pointer to binding data](#pointer-to-binding-data)) is protected by RBAC then the service consumer must pass a Service Account in its `ServiceBinding` CR to allow implementations of this specification to fetch information from that Secret.  If the implementation already has access to all resources in the cluster (as is the case with the Service Binding Operator) it must ensure it uses the provided Service Account instead of its own - blocking the bind if a Service Account was needed (according to the binding data) but not provided.

Example of a partial CR:

```
 services:
    - group: postgres.dev
      kind: Service
      resourceRef: user-db
      version: v1beta1
      serviceAccount: <my-sa>
```

Scenario 3 can also be enforced by RBAC in a similar fashion, but passing the service account in the `application` section of the `RequestBinding` CR.

#### Subscription-based services

There are a variety of service providers that require a subscription to be created before accessing the service. Examples:
* an API management framework that provides apiKeys after a plan subscription has been approved
* a database provisioner that spins single-tenant databases upon request
* premium services that deploy a providers located in the same physical node as the caller for low latency
* any other type of subscription service

The only requirement from this specification is that the subscription results in a k8s resources (Secret, etc), containing a partial or complete set of binding data (defined in [Service Binding Schema](#service-binding-schema)).  From the `ServiceBinding` CR's perspective, this resource looks and feels like an additional service.

Example of a partial CR:

```
 services:
    - group: postgres.dev
      kind: Service
      resourceRef: global-user-db
      version: v1beta1
    - group: postgres.dev
      kind: Secret
      resourceRef: specific-user-db
      version: v1beta1      
```


### Mounting and injecting binding information

This specification allows for data to be mounted using volumes or injected using environment variables.  The best practice is to mount any sensitive information, such as passwords, since that will avoid accidently exposure via environment dumps and subprocesses. 

The decision to mount vs inject is made in the following ascending order of precedence:
* value of the `bindAs` attribute in the backing service as defined in its [annotations](annotations.md#data-model--building-blocks-for-expressing-binding-information), applying to the binding item referenced by the annotation.
* value of `ServiceBinding`'s global `bindAs` element, which applies to all binding data.
* value of the `bindAs` attribute in each of the `dataMappings` elements inside `ServiceBinding`.

#### Injecting data

The key `SERVICE_BINDINGS` acts as a global map of the service bindings and **MUST** always be injected into the environment.  It contains a JSON payload with a simple map, where the key points to an available service binding item and the value equals either `envVar` or `volume:<path>`.  Example:

```
{ "KAFKA_USERNAME": "envVar", "KAFKA_PASSWORD": "volume:/platform/bindings/secret/" }
```

In the example above, the application can query the environment variable `SERVICE_BINDINGS`, walk its JSON payload and learn that `KAFKA_USERNAME` is also available as an environment variable, and that `KAFKA_PASSWORD` is avallable as a mounted file inside the directory `/platform/bindings/secret/`.


#### Mounting data
Implementations of this specification must bind the following data into the consuming application container:

```
<mountPathPrefix>/bindings/metadata/<persisted_configMap>
<mountPathPrefix>/bindings/request/<ServiceBinding_CR>
<mountPathPrefix>/bindings/secret/<persisted_secret>
```

Where:
* `<mountPathPrefix>` defaults to `platform` if not specified in the `ServiceBinding` CR via the `mountPathPrefix` element.
* `<persisted_configMap>` represents a set of files where the filename is a ConfigMap key and the file contents is the corresponding value of that key.  This is optional, as the ConfigMap is not mandatory.
* `<ServiceBinding_CR>` represents the requested `ServiceBinding` CR.
* `<persisted_secret>` represents a set of files where the filename is a Secret key and the file contents is the corresponding value of that key.


### Extra:  Consuming binding

*  How are application expected to consume binding information 
*  Each framework may take a different approach, so this is about samples & recommendations (best practices)
*  Validates the design
