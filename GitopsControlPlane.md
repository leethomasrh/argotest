# Managing applications via a Gitops Control Plane

This document provides steps to install the Openshift-Gitops Operator and configure one instance of ArgoCD that can be used as a cluster Gitops Control Plane to deploy applications into any namespace that is managed by ArgoCD.
 
This example:

- installs the openshift-gitops operator
- configures a GitOps control plane instance
- creates four ArgoCD managed namespaces
- creates an ArgoCD Admin user
- creates an ArgoCD Audit/Read only user
- creates two developer accounts limited via both OCP RBAC and ArgoCD project/RBAC
- Deploys an example application using an app of apps pattern in namespace app-one
- Deploys multiple artifacts using kustomize and an ArgoCD project with multiple destinations namespaces.  This demonstrates the ability of a development team to manage OCP objects in multiple namespaces from one Git repo.
 
This document will focus on doing everything via yaml files which will support teams wishing to configure day 2 items via a GitOps methodology.  Many of the tasks discussed can be accomplished via the OCP console as well.

## Installing the Openshift-Gitops Operator
1. Create the operator subscription

    ```yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: openshift-gitops-operator
      namespace: openshift-operators
    spec:
      channel: stable
      installPlanApproval: Automatic
      name: openshift-gitops-operator
      source: redhat-operators
      sourceNamespace: openshift-marketplace
      startingCSV: openshift-gitops-operator.v1.4.2
    ```

## Creating the identity provider, users, namespaces, groups and rolebindings

1. [Add an Identity provider if you haven't already got one configured](https://docs.openshift.com/container-platform/4.7/authentication/identity_providers/configuring-htpasswd-identity-provider.html)
  
      In this example I'm using an htpasswd identity provider for simplicity.
2. Add the user accounts
    
   ##### First developer account
    ```yaml
    apiVersion: user.openshift.io/v1
    groups: null
    identities:
    - htpasswd:app-one
    kind: User
    metadata:
      name: app-one
    ```

   ##### Second developer account

    ```yaml
    apiVersion: user.openshift.io/v1
    groups: null
    identities:
    - htpasswd:app-two
    kind: User
    metadata:
      name: app-two
    ```
   ##### Argo CD Audit account
   ##### granted read only permissions in ArgoCD

    ```yaml
    apiVersion: user.openshift.io/v1
    groups: null
    identities:
    - htpasswd:argoaudit
    kind: User
    metadata:
      name: argoaudit   
    ```
   ##### Argo CD Admin account
   ##### granted admin rights in ArgoCD and the Gitops Control Plane namespace

    ```yaml
    apiVersion: user.openshift.io/v1
    groups: null
    identities:
    - htpasswd:devops
    kind: User
    metadata:
      name: devops
    ```

3. Create the namespaces
   ##### devops namespace
   ##### This namespace will be used for the  Gitops Control Plane
   
   ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
    labels:
      kubernetes.io/metadata.name: devops
    name: devops
    ```
    
   ##### 1st Development team namespace: app-one
    
    Important considerations for the files below:
    * the label argocd.argoproj.io/managed-by needs to be set to the namespace the Gitops control plane instance will be running in.
    
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
    labels:
      kubernetes.io/metadata.name: app-one
      argocd.argoproj.io/managed-by: devops
    name: app-one
    ```

   ##### 2nd Development team namespaces: argotest21, argotest31, argotest41

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      labels:
        argocd.argoproj.io/managed-by: devops
        kubernetes.io/metadata.name: argotest21
      name: argotest21
    spec: {}
    ```
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      labels:
        argocd.argoproj.io/managed-by: devops
        kubernetes.io/metadata.name: argotest31
      name: argotest31
    spec: {}
    ```
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      labels:
        argocd.argoproj.io/managed-by: devops
        kubernetes.io/metadata.name: argotest41
      name: argotest41
    spec: {}
    ```

4. Create the groups

   ##### argocdadmins
   ##### Anyone assigned to this group will have admin access in ArgoCD

    ```yaml
    apiVersion: user.openshift.io/v1
    kind: Group
    metadata:
      name: argocdadmins
    users:
    - devops
    ```

   ##### argocdaudit
   ##### Anyone assigned to this group will have readonly access in ArgoCD

    ```yaml
    apiVersion: user.openshift.io/v1
    kind: Group
    metadata:
      name: argocdaudit
    users:
    - argoaudit 
    ```

   ##### app-one
   ##### Assigned users will have ArgoCD app-one AppProject RBAC access

    ```yaml
    apiVersion: user.openshift.io/v1
    kind: Group
    metadata:
      name: argocd-app-one
    users:
    - app-one
    ```

   ##### app-two
   ##### Assigned users will have ArgoCD app-two AppProject RBAC access

    ```yaml
    apiVersion: user.openshift.io/v1
    kind: Group
    metadata:
      name: argocd-app-two
    users:
    - app-two
    ```

5. Create rolebindings giving accounts the admin role in thier respective namespaces

   ##### Gitops Control Plane admin
   ##### Assigned to the devops user

    ```yaml
    kind: RoleBinding
    metadata:
      name: admin
      namespace: devops
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: admin
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: devops
    ```

   ##### admin role: app-one
   ##### Assigned to the app-one user

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: admin
      namespace: app-one
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: admin
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: app-one
    ```

   ##### admin role: argotest21, argotest31, argotest41
   ##### Assigned to the app-two user
  
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: admin
      namespace: argotest21
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: admin
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: app-two
    ```
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: admin
      namespace: argotest31
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: admin
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: app-two
    ```
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: admin
      namespace: argotest41
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: admin
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: app-two
    ```
## Creating the Gitops Control Plane ArgoCD instance
This step create the Gitops Control Plane ArgoCD instance.  This instance is the one that the Devops team will use to support Development teams.  The route created in the devops namespace will provide access to the ArgoCD GUI.

NOTE: The argocd instance automatically creates the default ArgoCD project.

1. Create the argocd instance running in the devops namespace.  

    Important considerations in the file below
    * You can choose to change the name of this instance to avoid confusion - this is the default value
      ```yaml
      metadata:
        annotations:
        name: argocd
      ```
    * RBAC definition grants the argocdadmins OCP group admin privileges in ArgoCD and the argocdaudit OCP group readonly privileges in ArgoCD
      ```yaml
        rbac:
          defaultPolicy: ""
          policy: \|
            g, system:cluster-admins, role:admin
            g, argocdadmins, role:admin
            g, argocdaudit, role:readonly
      ```
    NOTE: This step takes a couple of minutes to complete

   ##### ArgoCD Control Plane Instance

   ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: ArgoCD
    metadata:
      annotations:
      name: argocd
      namespace: devops
    spec:
      controller:
        processors: {}
        resources:
          limits:
            cpu: "2"
            memory: 2Gi
          requests:
            cpu: 250m
            memory: 1Gi
        sharding: {}
      dex:
        openShiftOAuth: true
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 250m
            memory: 128Mi
      grafana:
        enabled: false
        ingress:
          enabled: false
        route:
          enabled: false
      ha:
        enabled: false
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 250m
            memory: 128Mi
      initialSSHKnownHosts: {}
      prometheus:
        enabled: false
        ingress:
          enabled: false
        route:
          enabled: false
      rbac:
        defaultPolicy: ""
        policy: \|
          g, system:cluster-admins, role:admin
          g, argocdadmins, role:admin
          g, argocdaudit, role:readonly
        scopes: '[groups]'
      redis:
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 250m
            memory: 128Mi
      repo:
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: 250m
            memory: 256Mi
      server:
        autoscale:
          enabled: false
        grpc:
          ingress:
            enabled: false
        ingress:
          enabled: false
        resources:
          limits:
            cpu: 500m
            memory: 256Mi
          requests:
            cpu: 125m
            memory: 128Mi
        route:
          enabled: true
        service:
          type: ""
      tls:
        ca: {}
   ```
    Installation has completed once the argocd object reports Running|Available|Success for all of the components under the status section of the output of the following command.

    ```bash
    oc describe argocd argocd -n devops
    ```

## Create the ArgoCD projects and associated RBAC permissions

This will create the ArgoCD projects and associated ArgCD RBAC settings.  These projects will limit the users in the associated OCP group to the specified namespace(s) and also determines what they can do.

1. Create the ArgoCD AppProject app-one object    

    Important considerations in the files below

    * the destination namespace(s) accessible by applications assigned to this app project
      ```yaml
      destinations:
      - name: '*'
        namespace: app-one
      ```
    * the OCP group a user must be a member of to access applications assigned to this app project
      ```yaml
      roles:
      - groups:
        - argocd-app-one
      ```
    * the ArgoCD RBAC settings for this app project
      ```yaml
        policies:
        - p, proj:app-one:app-one, applications, *, app-one/*, allow
      ```

   ##### AppProject app-one

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
      name: app-one
      namespace: devops
    spec:
      clusterResourceWhitelist:
      - group: '*'
        kind: '*'
      destinations:
      - name: '*'
        namespace: app-one
        server: '*'
      namespaceResourceWhitelist:
      - group: '*'
        kind: '*'
      roles:
      - groups:
        - argocd-app-one
        name: app-one
        policies:
        - p, proj:app-one:app-one, applications, *, app-one/*, allow
      sourceRepos:
      - '*'
    ```
2. Create the ArgoCD AppProject app-two object

   ##### AppProject app-two

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata:
      name: app-two
      namespace: devops
    spec:
      clusterResourceWhitelist:
      - group: '*'
        kind: '*'
      destinations:
      - name: '*'
        namespace: argotest21
        server: '*'
      - name: '*'
        namespace: argotest31
        server: '*'
      - name: '*'
        namespace: argotest41
        server: '*'
      namespaceResourceWhitelist:
      - group: '*'
        kind: '*'
      roles:
      - groups:
        - argocd-app-two
        name: app-two
        policies:
        - p, proj:app-two:app-two, applications, *, app-two/*, allow
      sourceRepos:
      - '*'
    ```

## Create the sample applications
### App of Apps pattern

This will create an instance of the sample application using an app of apps pattern.  The root app points at a Git repository that contains a yaml definition for child application object(s).  This child application object then deploys the sample application.

Using the app of apps pattern simplifies the application deployment process by:
* The development team creates all the necessary application artifacts within the child application repo
* Allows the Devops team to create a standardized/automated application onboarding process
* Provides a clear demarcation point between infrastructure and application management
* Enables the Devops team to manage multiple child applications from a single root application object

NOTE: The child application object still needs to be created in the Gitops Control Plane namespace.

An account with permissions in the Gitops Control Plane namespace will need to create the root app which, once the child applicaiton object has been committed to the child application git repo, in turn creates the child app.  

The Devops team will need to refresh the root application object if:

* The root application is created before the child application yaml object has been committed to the child application repo

* The Development team creates new child application objects 

* The existing child applicaiton object is deleted and recreated/restored

Once the Deovps team creates the child app object(s) the development team can take over management of the child application object(s).

#### Example App of Apps deployment

IMPORTANT: It is not recommended to give developers access to create objects in the Gitops Control Plane namespace.

In this example the root application will be created to automatically sync, but the child application will be created to be synchronized manually.  This is useful to allow the Development team control of the intial application deployment; however, since the child application object is managed by the developement team they can choose to have it automatically sync at any time.

1. login as the devops user
        
      ```bash
      oc login https://<cluster api URL>:6443 -u devops
      ```

2. Create the root application object

    Important considerations in the file below:

    * The app project this application will be assigned to
    ```yaml
          project: app-one
    ```
    * The syncPolicy setting for the root application 
    ```yaml
      syncPolicy:
        automated: {}
    ```
    * The path to the application manifests within the child application Git repo
    * URL for the child application git repo
    * The targetRevision to use.
   
    ```yaml
      source:
        path: .
        repoURL: https://github.com/leethomasrh/argotest
        targetRevision: HEAD
    ```

   ##### root application in the app-one ArgoCD project

   ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: root-app
      namespace: devops
    spec:
      destination:
        namespace: app-one
        server: https://kubernetes.default.svc
      project: app-one
      syncPolicy:
        automated: {}
      source:
        path: .
        repoURL: https://github.com/leethomasrh/argotest
        targetRevision: HEAD
   ```
3. Create the child application object

    Important considerations in the file below:
    * The metadata namespace needs to be the Gitops Control Plane namespace
    * The destination namespace needs to be the Development namespace
    ```yaml
    metadata:
      name: sample-app
      namespace: devops
    spec:
      destination:
        namespace: app-one
    ```
    * The app project for the child application needs to match the root application
    ```yaml
          project: app-one
    ```

   ##### child application in the app-one ArgoCD project

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: sample-app
      namespace: devops
    spec:
      destination:
        namespace: app-one
        server: https://kubernetes.default.svc
      project: app-one
      source:
        path: app
        repoURL: https://github.com/redhat-developer/openshift-gitops-getting-started
        targetRevision: HEAD
    ```
  Creation of the root application object can be included in the application onboarding automation (creating the users, groups, namespaces, ArgoCD application projects, root application object) and simply refreshed when the Development team has created the child application object(s) and is ready to begin testing.  

### Deploying objects in multiple namespaces

IMPORTANT: It is not recommended to give developers access to create objects in the Gitops Control Plane namespace. 

This example will show how a development team can manage objects in multiple OCP namespaces.  For simplicity this example does not use the app of apps pattern but can be modified to use it if desired.

An ArgoCD Application Project with multiple destination namespaces assigned will be created.  Additionally, the development users in the group associated with this Application Project need to have the necessary permissions in all the OCP namespaces and in the ArgoCD Application Project RBAC.  

NOTE: It is important that all the objects created in this example specify a namespace in the object yaml definition.  If a namespace is not defined the object will be created in the namespace used when the Application Project object is created (in this example the argotest21 namespace).  

This example creates an empty configmap object in argotest21, deploys a hello world application in argotest31, and attempts to create an empty configmap object in argotest41.  Hoever, the developer forgot to specify the namespace in the test-41 configmap so the test-41 configmap is created in the argotest21 namespace as specified in the ArgoCD App Project object.

1. login as the devops user
    ```bash
    oc login https://<cluster api URL>:6443 -u devops
    ```

2. Create the application object 

    Important considerations in the file below:
    * The metadata namespace needs to be the Gitops Control Plane namespace
    * The destination namespace needs to be the Development namespace
      ```yaml
      metadata:
        name: sample-app-2
        namespace: devops
      spec:
        destination:
          namespace: argotest21
      ```
    * The default namespace for the app project.  If a namespace isn't specified in an artifacts' yaml this is the namespace the artifact will be created in.
      ```yaml
      spec:
        destination:
          namespace: argotest21
      ```
    * The app project this application will be assigned to
      ```yaml
      project: app-two
      ```
    * The path within the development repository where the application artifacts are stored
    * The development repository URL
      ```yaml
        source:
          path: test
          repoURL: https://github.com/leethomasrh/argotest
      ```

   ##### application managing objects in multiple development namespaces

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: sample-app-2
      namespace: devops
    spec:
      destination:
        namespace: argotest21
        server: https://kubernetes.default.svc
      project: app-two
      source:
        path: test
        repoURL: https://github.com/leethomasrh/argotest
        targetRevision: HEAD
    ```
    The following artifacts are created in the development git repository referenced in the application object above.

   ##### kustomization.yaml

      ```yaml
      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization

      resources:
        - app1/configmap.yaml
        - app2/replicaset.yaml
        - app3/configmap.yaml
      ```
      
   ##### empty configmap object in the argotest21 namespace

      ```yaml
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: test-21
        namespace: argotest21
      ```

   ##### hello-openshift application in the argotest31 namespace

      ```yaml
      apiVersion: v1
      kind: ReplicationController
      metadata:
        name: frontend-1
        namespace: argotest31
      spec:
        replicas: 1  
        selector:    
          name: frontend
        template:    
          metadata:
            labels:  
              name: frontend 
          spec:
            containers:
            - image: openshift/hello-openshift
              name: helloworld
              ports:
              - containerPort: 8080
                protocol: TCP
            restartPolicy: Always
      ```

   ##### empty configmap intended to be created in the argotest41 namespace
   ##### but creates in the argotest21 namespace
    
    NOTE: the namespace is intentionally not defined to demonstrate the app project default namespace

      ```yaml
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: test-41
      
      ```

## Additional Topics
### Developers creating ArgoCD projects to access other namespaces

In this example the ArgoCD project RBAC provides an additional layer of permissions which prevent non-admin users from creating App Project objects to access other namespaces.  If a non-admin user tries to create a project in the ArgoCD GUI the following error occurs.
```
Description
Unable to create project: permission denied: projects, create, Test, sub: CiRiMjZmMjdiZC1hNWQzLTQ3YWMtYWU0Ny1hNjVjOTk4YTk0MDgSCW9wZW5zaGlmdA, iat: 2022-02-17T17:49:33Z
```
## References
The information used to create this example is available here:

- [Setting up Gitops Control Plane instance](https://developers.redhat.com/articles/2021/08/03/managing-gitops-control-planes-secure-gitops-practices#application_delivery_with_openshift_gitops)
- [Configure Argo RBAC with Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/#configuring-rbac-with-projects)
- [Configure Argo RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/#rbac-permission-structure)
- [Example app used](https://github.com/redhat-developer/openshift-gitops-getting-started)
