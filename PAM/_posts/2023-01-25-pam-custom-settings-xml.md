---
layout: post_with_sidebar
title: Use Custom settings.xml on PAM deployed on OpenShift
date: 2023-01-25 12:25
category: PAM
subcategory: OpenShift

tags: [PAM, OpenShift, HowTo]
 
---

In restricted network environments, the access to public Maven repositories such https://maven.repository.redhat.com and https://repo1.maven.org may not be available without a proxy and in older PAM versions there's no way to specify it directly using configurations. This post explains how to inject a custom Maven settings.xml inside the pods.

- Create the `configmap` as follows based on the `settings.xml` file for Maven

~~~  bash
$ oc create cm maven-settings-config --from-file settings.xml
~~~

- Edit the `configmap` that matches the product version being deployed. *For example, for 7.9.0*

{%- capture note -%}This configmap contains templates that the operator uses to deploy KieServer and Decision Central or Business Central, so that is where a volumeMount and a volume will be configured that will allow mounting the secret in the mountPath.{%- endcapture -%}

{%- include note.html content=note -%}

~~~ bash
$ oc edit cm kieconfigs-7.9.0
~~~

~~~ yaml
console:
  deploymentConfigs:
    - metadata:
        name: "[[.ApplicationName]]-[[.Console.Name]]"
      ...
      spec:
        ...
        template:
        ...
          spec:
          ...
            containers:
              - name: "[[.ApplicationName]]-[[.Console.Name]]"
              ...
                volumeMounts:
                  - name: "maven-settings-volume" 
                    mountPath: "/data/maven"
                    readOnly: true
            ...
            volumes:
                - configMap:
                    defaultMode: 420
                    name: maven-settings
                name: maven-settings-volume

                  ...
~~~

~~~ yaml
servers:
  ## RANGE BEGINS
  #[[ range $index, $Map := .Servers ]]
  ## KIE server deployment config BEGIN
  - deploymentConfigs:
      - metadata:
          name: "[[.KieName]]"
        ...
        spec:
          ...
          template:
           ...
            spec:
             ...
              containers:
                - name: "[[.KieName]]"
                  ...
                  volumeMounts:
                    - name: "maven-settings-volume" 
                      mountPath: "/data/maven"
                      readOnly: true
                      ...
              volumes:
                  - configMap:
                      defaultMode: 420
                      name: maven-settings
                    name: maven-settings-volume
~~~

- In the `kieapp yaml` the following settings must be configured:

~~~ yaml
apiVersion: app.kiegroup.org/v2
kind: KieApp
metadata:
...
spec:
  objects:
    console:
      ...
      env:
        - name: MAVEN_SETTINGS_XML
          value: /data/maven/settings.xml
      ...
    servers:
      - name: MAVEN_SETTINGS_XML
        value: /data/maven/settings.xml
      ...

        
~~~

Some considerations:

- The yaml used for deployment is not affected. i.e. KieApp resource remains the same.
- Changes affect the templates used by the operator to create the deployment configs for Kie Server and Business Central
- Changes to the templates will not be affected by operator upgrades.
- Changes to the templates must be propagated when upgrading to a new product release, such as 7.9.1. Take care because templates might change between different versions.
- Since this configuration replaces the whole settings.xml generated when the container starts, if Business Central is used as an artifact repository for kjars, the settings.xml must contain the Business Central address too.

