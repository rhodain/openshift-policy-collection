### VALUES EXPLANATION
Values pro jednotlive clustery jsou skladany z nekolika dilcich values dohromady. Priklad skladani Values v nasledujici tabulce:
|Priorita|Path|Popis|
|:---|:---|:---|
|2|`apps/{HELM_APPLICATION_NAME}/values.yaml`|Defaultni Values Chartu - sdilene pro vsechny klastry|
|1|`apps/values/{HELM_APPLICATION_NAME}.yaml`|Doplnkove Values k Chartu - sdilene pro vsechny klastry|
|NA|`cluster-configs/{CLUSTER_NAME}/config-cluster.conf`|Globalni Values pro dany klastr + specifikace `components`, tzn jake aplikace budou na klastru nasazeny|
|0|`cluster-configs/{CLUSTER_NAME}/config-{HELM_APPLICATION-NAME}.conf`|Doplnkove Values k Chartu - values pro dany klastr|

>__NOTE__: Prepisovani Values hodnot by melo byt podle priority uvedene v tabulce. Nizsi cislo = vyssi priorita. 

```
├── apps
│   ├── namespace (HELM application)
│   │   ├── Chart.yaml
│   │   ├── charts
│   │   ├── templates
│   │   └── values.yaml
│   └── values
│       ├── namespace.yaml
│       ├── secrets-testing-app.yaml
├── cluster-configs
│   ├── lab5
│   │   ├── config-cluster.yaml
│   │   ├── config-namespace.yaml
│   │   └── config-secrets-testing-app.yaml
│   └── local-cluster
│       ├── config-cluster.yaml
│       ├── config-namespace.yaml
│       ├── config-sealed-secrets.yaml
│       ├── config-secrets-db.yaml
│       └── config-secrets.yaml
```

#### CLUSTER VALUES
Priklad Values pro jeden klastr. (`config-cluster.yaml`) <BR>
```
---
# -----------------------------------
# GLOBAL SETTINGS
# -----------------------------------
type: vmware
baseDomain: lab5.vs.csint.cz
environment: lab
clusterName: lab5

proxy:
  http_proxy: "http://ngproxy-test.csint.cz:8080/"
  https_proxy: "http://ngproxy-test.csint.cz:8080/"
  no_proxy: ".cluster.local,.cs-test.cz,.cs.cz,.csin.cz,.csint.cz,.svc,10.88.88.0/21,10.88.88.0/24,100.123.0.0/17,100.123.128.0/17,127.0.0.1,api-int.lab5.ocp4.vs.csint.cz,localhost"


# -----------------------------------
# COMPONENTS
# -----------------------------------
components:
```
Helm aplikace
- `appModel` Key urcuje, jaky mode [pull|push] bude pro aplikaci pouzit. Pokud je `appModel=push`, aplikace bude v push modu. V opacnych pripadech (appModel="", appModel neni specifikovan) bude aplikace v pull modu. Zamerem je pouzivat push mode pouze pro local-cluster (HUB).
```
  - appName: "namespace"
    appType: helm
    appNamespace: ""
    appVersion: "lpechacek/dev"
    appReleaseName: namespace
    appRepoURL: https://github.com/csas-ops/openshift-policy-collection.git
    appRepoPath: openshift-clusters/apps/namespace
    appSyncPolicy:
      automated:
        selfHeal: true
        prune: false
      syncOptions:
        - CreateNamespace=false
        - SkipDryRunOnMissingResource=true
        - Wait=true
    appAnnotation:
      example.com/test-annotation: "This is a test annotation"
      example.com/test-annotation-2: "This is a test annotation 2"
    appLabels:
      purpose: example
    appModel: push
```
Helm aplikace vyuzivajici secret z `secret-db` aplikace. 
- Vyzadani secretu se deje na zaklade specifikace Key `appSecrets`. Jde o list secretu pro danou aplikaci. Secret musi existovat v `secrets-db`.
- Pokud je Key `appSecrets` specifikovan, cluster-configurations-secrets applicationset vytvori policy a ta nasledne nahraje secret na pozadovany klastr.
- Pri odstraneni secretu z listu NENI secret smazan z klatru. Jedna se o "zadouci" vlastnost policy. Udajne jako ochrana pred nechtenym smazanim.
```
  - appName: "secrets-testing-app"
    appType: helm
    appNamespace: "secrets-testing-app"
    appVersion: "lpechacek/dev"
    appReleaseName: secrets-testing-app
    appRepoURL: https://github.com/csas-ops/openshift-policy-collection.git
    appRepoPath: openshift-clusters/apps/secrets-testing-app
    appSecrets:
    - test-secret-3.yaml
    appSyncPolicy:
      automated:
        selfHeal: true
        prune: false
      syncOptions:
        - CreateNamespace=false
        - SkipDryRunOnMissingResource=true
        - Wait=true
```
Helm aplikace u ktere je treba provest postrender
- Vyzadani postrenderu se deje na zaklade Key `appPostRender`.
- Key appPostRender vola predem vytvoreny helm ktery provede zmeny. Lze provest i nastaveni jinym zpusobem, ne jen pres HELM. Helm byl zvolen pro moznost Kustomizace post renderu pro jednotlive klastry.
```
  - appName: "sealed-secrets"
    appType: helm
    appNamespace: "sealed-secrets"
    appVersion: "lpechacek/dev"
    appReleaseName: sealed-secrets
    appRepoURL: https://github.com/csas-ops/openshift-policy-collection.git
    appRepoPath: openshift-clusters/apps/sealed-secrets
    appSyncPolicy:
      automated:
        selfHeal: true
        prune: false
      syncOptions:
        - CreateNamespace=false
        - SkipDryRunOnMissingResource=true
        - Wait=true
    appPostRender:
      repoURL: https://github.com/csas-ops/openshift-policy-collection.git
      releaseName: sealed-secrets-postrender
      appVersion: "lpechacek/dev"
      repoPath: openshift-clusters/apps/sealed-secrets-postrender
```

### APPSET - secrets
Distribuce secretu ulozenych v aplikaci `secrets-db` v podobe sealedsecret

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: clusters-configurations-secrets
  namespace: openshift-gitops
spec:
```
```
  goTemplate: true                              #Enabling Go text templating => We can use Values
  goTemplateOptions: ["missingkey=error"]       #Pokud je volana nedefinovana hodnota v Template, je reportovan Error misto tiche ignorace.
```
[DOC: ArgoCD Go template](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/GoTemplate/) <BR>
[DOC: Generators](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/)
```
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/csas-ops/openshift-policy-collection.git
              revision: "lpechacek/dev"
              files:
                - path: "openshift-clusters/cluster-configs/**/config-cluster.yaml"
          - list:
              elementsYaml: |
                {{- range .components }}                            # Filtrace aplikaci - vyber pouze aplikaci, ktere maji definovane .appSecrets
                {{- if hasKey . "appSecrets" }}
                - elementName: {{ .appName }}
                  elementNamespace: {{ .appNamespace }}
                  elementSecrets: {{ toJson .appSecrets }}
                  elementSyncPolicy: {{ toJson .appSyncPolicy }}
                {{- end }}
                {{- end }}
```
```
  template:
    metadata:
      name: '{{.clusterName}}-{{.elementName}}-secrets'
      labels:
        environment: '{{.environment}}'
        cluster: '{{.clusterName}}'
        type: '{{.type}}'
    spec:
      project: openshift
      source:
        repoURL: https://will-be-replaced
      destination:
        name: "local-cluster"
        namespace: policies
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=false
          - SkipDryRunOnMissingResource=true
          - Wait=true
          - PrunePropagationPolicy=foreground
```
**Patchovani `.spec.template`. Slozitejsi templating lze pouzit pouze v `spec.templatePatch`. V default `.spec.template` nelze slozitejsi templating pouzit.**
```
  templatePatch: |
    spec:
      source: null
      sources:
        - repoURL: https://github.com/csas-ops/openshift-policy-collection.git
          targetRevision: 'lpechacek/dev'
          path: openshift-clusters/apps/secrets-db
          helm:
            releaseName: '{{.elementName}}-secrets-distribution'
            valuesObject:
              renderOnlyPolicy: true
              secretForApp: {{ .elementName }}
              secretNamespace: {{ .elementNamespace }}
              {{- with .elementSecrets }}
              secretsList:
              {{- toYaml . | nindent 16 }}
              {{- end }}
            valueFiles:
              - "/openshift-clusters/apps/values/secrets-db.yaml"
              - "/openshift-clusters/apps/values/{{.elementName}}.yaml"
              - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-{{.elementName}}.yaml"
              - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-cluster.yaml" 
```
**Pridani syncPolicy do template a tim i do vygenerovane argocd aplikace.**
```
      {{/* 
      ------------------------------------------------------------------------
      SYNC POLICY
      ------------------------------------------------------------------------
       */}}

      {{- if .elementSyncPolicy }}
      syncPolicy:
        {{- if .elementSyncPolicy.automated }}
        {{- with .elementSyncPolicy.automated }}
        automated:
        {{- toYaml . | nindent 10 }}
        {{- end }}
        {{- end }}
        {{- if .elementSyncPolicy.syncOptions }}
        syncOptions:
          - PrunePropagationPolicy=foreground
        {{- range .elementSyncPolicy.syncOptions }}
          - {{ . }}
        {{- end }}
        {{- end }}
      {{- end }}
```

### APPSET - application
```
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: clusters-configurations
  namespace: openshift-gitops
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/csas-ops/openshift-policy-collection.git
              revision: "lpechacek/dev"
              files:
                - path: "openshift-clusters/cluster-configs/**/config-cluster.yaml"
          - list:
              elementsYaml: '{{ .components | toJson }}'
  template:
    metadata:
      name: '{{.clusterName}}-{{.appName}}'
    spec:
      project: openshift
      source:
        repoURL: https://will-be-replaced
      destination:
        name: '{{.clusterName}}'
        namespace: '{{.appNamespace}}'
      syncPolicy:
        automated:
          selfHeal: true
          prune: false
        syncOptions:
          - CreateNamespace=false
          - SkipDryRunOnMissingResource=true
          - Wait=true
```
**Patchovani `.spec.template`. Slozitejsi templating lze pouzit pouze v `spec.templatePatch`. V default `.spec.template` nelze slozitejsi templating pouzit.**
- Patchovani aplikace je rozdeleno podle toho, zda je treba single source nebo multisource aplikace. Pokud mame nastaveno `.Values.components.{item}.appPostRender`, jedna se o multisource aplikaci.
- Patchovani je dale rozdeleno podle typu aplikace (`.Values.components.{item}.appType`)
- Pokud mame Helm Chart, ktery byl vytvoren treti stranou a potrebujeme pridat dalsi resources, lze pouzit postrender pomoci nastaveni `.Values.components.{item}.appPostRender`.
```
  templatePatch: |
      {{/* 
      ----------------------------------------------------------------------------
      SETTING PULL/PUSH MODE OF APPLICATION
      ----------------------------------------------------------------------------
      */}}
      
      {{- $ArgoAppModel := "" }}
      {{- if and (hasKey . "appModel") ( eq .appModel "push" ) }}
      {{- $ArgoAppModel = "push" }}
      {{- else }}
      {{- $ArgoAppModel = "pull" }}
      {{- end }}

      {{/* 
      ----------------------------------------------------------------------------
      UPDATING METADATA LABELS & ANNOTATIONS BASED ON APPLICATION MODE [pull|push]
      ----------------------------------------------------------------------------
      */}}

      metadata:
        name: '{{.clusterName}}-{{.appName}}'

        {{- if and ( hasKey . "appAnnotation" ) .appAnnotation }}
        annotations:
          {{- toYaml .appAnnotation | nindent 4 }}
          {{- if eq $ArgoAppModel "pull" }}
          apps.open-cluster-management.io/ocm-managed-cluster: "{{.clusterName}}"
          apps.open-cluster-management.io/ocm-managed-cluster-app-namespace: "{{.sysArgoNamespace}}"
          argocd.argoproj.io/skip-reconcile: "true"
          {{- end }}
        
        {{- else if or (and (hasKey . "appAnnotation" ) (not (.appAnnotation)) ) (not (hasKey . "appAnnotation")) }}
          {{- if eq $ArgoAppModel "pull" }}
        annotations:
          apps.open-cluster-management.io/ocm-managed-cluster: "{{.clusterName}}"
          apps.open-cluster-management.io/ocm-managed-cluster-app-namespace: "{{.sysArgoNamespace}}"
          argocd.argoproj.io/skip-reconcile: "true"
          {{- end }}
        {{- else }}
        {{- end }}      
        
        labels:
          {{- if and ( hasKey . "appLabels" ) .appLabels }}
          {{- toYaml .appLabels | nindent 4 }}
          {{- end }}
          {{- if eq $ArgoAppModel "pull" }}
          apps.open-cluster-management.io/pull-to-ocm-managed-cluster: "true"
          {{- end }}
          environment: '{{.environment}}'
          cluster: '{{.clusterName}}'
          type: '{{.type}}'


      {{/* 
      ----------------------------------------------------------------------------
      UPDATING SPEC SECTION
      ----------------------------------------------------------------------------
      */}}    
      spec:
        
        {{/* 
        --------------------------------------------------------------------------
        SINGLE SOURCE
        --------------------------------------------------------------------------
        */}}

        {{- if not ( hasKey . "appPostRender") }}
        source:
          {{/*
          ------------------------------------------------------------------------
          SINGLE SOURCE - HELM
          ------------------------------------------------------------------------
          */}}

          {{- if eq "helm" .appType }}
            repoURL: '{{.appRepoURL}}'
            targetRevision: '{{.appVersion}}'
            path: '{{.appRepoPath}}'
            helm:
              releaseName: '{{.appReleaseName}}'
              valueFiles:
                - "/openshift-clusters/apps/values/{{.appName}}.yaml"
                - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-{{.appName}}.yaml"
                - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-cluster.yaml"
          {{- end }}
          
          {{/*
          ------------------------------------------------------------------------
          SINGLE SOURCE - KUSTOMIZE
          ------------------------------------------------------------------------
          */}}

          {{- if eq "kustomize" .appType }}
            repoURL: '{{.appRepoURL}}'
            path: '{{.appRepoPath}}'
            targetRevision: '{{.appVersion}}'
          {{- end }}

        {{- else }}
        
        {{/* 
        --------------------------------------------------------------------------
        MULTI SOURCE
        --------------------------------------------------------------------------
        */}}

        source: null
        sources:
          {{/*
          ------------------------------------------------------------------------
          MULTI SOURCE - HELM
          ------------------------------------------------------------------------
          */}}

          {{- if eq "helm" .appType }}
          - repoURL: '{{.appRepoURL}}'
            targetRevision: '{{.appVersion}}'
            path: '{{.appRepoPath}}'
            helm:
              releaseName: '{{.appReleaseName}}'
              valueFiles:
                - "/openshift-clusters/apps/values/{{.appName}}.yaml"
                - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-{{.appName}}.yaml"
                - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-cluster.yaml"

          {{- end}}

          {{/*
          ------------------------------------------------------------------------
          MULTI SOURCE - KUSTOMIZE
          ------------------------------------------------------------------------
          */}}

          {{- if eq "kustomize" .appType }}
            repoURL: '{{.appRepoURL}}'
            path: '{{.appRepoPath}}'
            targetRevision: '{{.appVersion}}'
          {{- end }}

          {{/*
          ------------------------------------------------------------------------
          MULTI SOURCE - POST-RENDER
          ------------------------------------------------------------------------
          */}}

          {{- if ( hasKey . "appPostRender" ) }}
          - repoURL: '{{.appPostRender.repoURL}}'
            targetRevision: '{{.appPostRender.appVersion}}'
            path: '{{.appPostRender.repoPath}}'
            helm:
              releaseName: '{{.appPostRender.releaseName}}'
              valueFiles:
                - "/openshift-clusters/apps/values/{{.appName}}.yaml"
                - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-{{.appName}}.yaml"
                - "/openshift-clusters/cluster-configs/{{.clusterName}}/config-cluster.yaml"     
          {{- end }}
          
        {{- end }}
```
**Pridani syncPolicy do template a tim i do vygenerovane argocd aplikace.**
```
      {{/* 
      ------------------------------------------------------------------------
      SYNC POLICY
      ------------------------------------------------------------------------
       */}}
      {{- if and (hasKey . "appSyncPolicy") .appSyncPolicy }}
      syncPolicy:
        {{- if and (hasKey .appSyncPolicy "automated") .appSyncPolicy.automated }}
        {{- with .appSyncPolicy.automated }}
        automated:
        {{- toYaml . | nindent 10 }}
        {{- end }}
        {{- end }}
        {{- if and (hasKey .appSyncPolicy "syncOptions") .appSyncPolicy.syncOptions }}
        {{- with .appSyncPolicy.syncOptions }}
        syncOptions:
        {{- toYaml . | nindent 10 }}
        {{- end }}
        {{- end }}
      {{- end }}
```

### APPSET TESTING
Po zmene appset je vhodne ho pred nasazenim otestovat. <BR>

**Login to HUB argocd**
```
argocd login openshift-gitops-server-openshift-gitops.apps.hub.ocp4.vs.csint.cz:443 --username admin --insecure
```
**Prepnuti do adresare**
- pred zadanim prikazu je treba jit do `openshift-clusters/appsets`
```
ls -la 
-rw-r--r--@  1 ext44363  staff  3193 Feb 28 09:34 clusters-configurations-secrets.yaml
-rw-r--r--@  1 ext44363  staff  7667 Feb 28 08:40 clusters-configurations.yaml
```
**Test appset**
```
argocd appset generate clusters-configurations-secrets.yaml -o yaml
```
- Vystupem je yaml.
- Pokud je spatna syntaxe v appset, yaml neni vygenerovan a je vracena chyba.
- Je treba zkontrolovat vystup, zda je validni!
- Pozor na spatne odsazeni pres `nindent`!

### APP secret-db

`/tmp/helm-values-secret.yaml` pro simulaci secret appsetu, ktery nastavuje nasledujici hodnoty podle hodnot ve values.
```
secretForApp: secrets-testing-app
secretNamespace: secrets-testing-app
secretsList:
- test-secret-3.yaml
- test-secret-4.yaml
renderOnlyPolicy: true
```

Testovani celeho chartu
```
helm template secrets-db . \
-f ../values/secrets-testing-app.yaml \
-f ../../cluster-configs/lab5/config-cluster.yaml \
-f ../../cluster-configs/lab5/config-secrets-testing-app.yaml \
-f ../values/secrets-db.yaml \
-f /tmp/helm-values-secret.yaml
```

Testovani pouze `template/policy.yaml`
```
helm template -s templates/policy.yaml . \
-f ../values/secrets-testing-app.yaml \
-f ../../cluster-configs/lab5/config-cluster.yaml \
-f ../../cluster-configs/lab5/config-secrets-testing-app.yaml \
-f ../values/secrets-db.yaml \
-f /tmp/helm-values-secret.yaml 
```

### PRIDANI NOVE APLIKACE
vysvetleni prioritizace/prepisovani Values souboru je uvedeno v ### VALUES EXPLANATION

<BR>

#### 1. ZDROJOVA APLIKACE
##### HELM
Pridani Helm Chartu do `openshift-clusters/apps/{app_name}` adresare pokud:
- Mame vlastni Helm Chart
- Mame Chart 3-ti strany ktery neni v zadnem repository
- Nechceme/nemuzeme instalovat chart z repository
##### KUSTOMIZE
Aktualne je aplikace instalovana pres Kustomize nahrana do `openshift-clusters/apps/{app_name}`, protoze nebyl prostup do `csas-dev`.

<BR>

#### 2. NASTAVENI SDILENYCH VALUES PRO APLIKACE
##### HELM
Je treba pridat sdilena values pro vsechny klastry do `openshift-clusters/apps/values/{app_name}.yaml`. Soubor se musi vytvorit ikdyz bude prazdny.

<BR>

#### 3. NASTAVENI VALUES PRO APLIKACI NA POZADOVANYCH KLASTRECH
##### HELM
Je treba vytvorit Value soubor pro danou aplikaci a pozadovany klastr v `openshift-clusters/cluster-configs/{cluster_name}/config-{app_name}.yaml`. Tyto Values jsou unikatni pro kazdy klastr.

<BR>

#### 4. POSTRENDER 
- Pokud vime, ze po instalaci aplikace bude treba dodelavat nejake resourcy, pripravime si helm chart `openshift-clusters/apps/{app_name}-postrender`. 
- Pro postrender byl zvolen HELM z duvodu mozne rozdilnosti pro jednotlive klastry.
- Jelikoz postrender je navazan na urcitou aplikaci, values pro postrender helm chart pridavame do values souboru pro urcitou aplikaci.
- pokud budou manifesty stejne pro vsechny klastr, muzeme do template dat primo tyto manifesty

Priklad:
|||
|:---|:---|
|instalovana aplikace| sealed-secrets|
|postrender|ano|
|postrender aplikace| sealed-secrets-postrender|
|values for postrender in| `openshift-clusters/cluster-configs/local-cluster/config-sealed-secrets.yaml`|

`openshift-clusters/cluster-configs/local-cluster/config-sealed-secrets.yaml` a values pro postrender
```
appPostRender:
  route:
    create: true
    name: certificate
    namespace: sealed-secrets
    hostname: certificate-sealed-secrets
    path: /v1/cert.pem
    labels:
      ingress-router: default
    tls:
      termination: edge
      insecureEdgeTerminationPolicy: Allow
```

<BR>

#### 5. SECRETS
Pokud nase aplikace potrebuje secrety, je treba tyto secrety pripravit. Secrety se ukladaji do  `openshift-clusters/apps/secrets-db/secrets/{cluster_name}/{secret_name}.yaml`. Dany soubor obsahuje sealedsecret. Je treba pridat `namespace: {cluster_name}-secrets` do souboru. Tim je zaruceno, ze secret bude vytvoren a namespace pro dany klastr.

**Priklad vytvoreni secretu test-tls-secret.yaml v aplikaci `secrets-db` pro klastr lab5:**
```
curl -k -o cert.pem https://certificate-sealed-secrets.apps.hub.ocp4.vs.csint.cz/v1/cert.pem

kubeseal -o yaml --cert cert.pem -f ./test-tls-secret.yaml --scope namespace-wide > {$PATH}/openshift-policy-collection/openshift-clusters/apps/secrets-db/secrets/lab5/test-tls
```
`test-tls-secret.yaml`
```
---
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  creationTimestamp: null
  name: test-tls-secret
  namespace: lab5-secrets
spec:
  encryptedData:
    tls.crt: . . . 
    tls.key: . . .
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/namespace-wide: "true"
      creationTimestamp: null
      name: test-tls-secret
    type: kubernetes.io/tls
```
Aby byl secret vytvoren ve spravnem namespace na HUB klastru, je treba pridat `namespace: {cluster_name}-secrets`.
`test-tls-secret.yaml`
```
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  annotations:
    sealedsecrets.bitnami.com/namespace-wide: "true"
  creationTimestamp: null
  name: test-tls-secret
  namespace: lab5-secrets
spec:
  encryptedData:
    tls.crt: . . . 
    tls.key: . . .
  template:
    metadata:
      annotations:
        sealedsecrets.bitnami.com/namespace-wide: "true"
      creationTimestamp: null
      name: test-tls-secret
      namespace: lab5-secrets           //PRIDANY RADEK
    type: kubernetes.io/tls
```

<BR>

#### 6. NASAZENI APLIKACE NA KLASTR
Mame-li splneni kroky 1-5, muzeme aplikaci nasadit na klastr. To se provede specifikaci aplikace v souboru pro klastr kde chceme aplikaci nasadit. Napr pro lab5 v `openshift-clusters/cluster-configs/lab5/config-cluster.yaml`.

Priklad:
`openshift-clusters/cluster-configs/lab5/config-cluster.yaml`
```
.
.
components:
  - appName: "sealed-secrets"
    appType: helm
    appNamespace: "sealed-secrets"
    appVersion: "lpechacek/dev"
    appReleaseName: sealed-secrets
    appRepoURL: https://github.com/csas-ops/openshift-policy-collection.git
    appRepoPath: openshift-clusters/apps/sealed-secrets
    appSyncPolicy:
      automated:
        selfHeal: true
        prune: false
      syncOptions:
        - CreateNamespace=false
        - SkipDryRunOnMissingResource=true
        - Wait=true
    appAnnotation:
      example.com/test-annotation: "This is a test annotation"
      example.com/test-annotation-2: "This is a test annotation 2"
    appLabels:
      example: customLabel 
    appModel: push                    #1
    appPostRender:                    #2
      repoURL: https://github.com/csas-ops/openshift-policy-collection.git
      releaseName: sealed-secrets-postrender
      appVersion: "lpechacek/dev"
      repoPath: openshift-clusters/apps/sealed-secrets-postrender
    appSecrets:                       #3
    - test-tls-secret.yaml            #4
```
\#1 - appModel: push se specifikuje pouze v pripade ze chceme aplikaci v push modu. Pro pull Key smazeme, pripadne nenastavime hodnotu. <BR>
\#2 - Post Render - duvod proc se Values pro postrender davaji do app values je svazani s aplikaci <BR>
\#3 - List secretu pozadovanych v aplikacnim namespace. Distribuce: sealedsecret -> HUB -> policy -> managed cluster <BR>
\#4 - Nazev souboru obsahujiciho sealedsecret

<BR>

#### 7. OTHER

Get info related to pull mode
```
oc get gitopscluster argo-acm-remoteclusters -o yaml
oc get placement lab2 -o yaml | yq
oc get placementdecisions.cluster.open-cluster-management.io lab2-decision-1 -o yaml | yq
```
Get info related to application
```
oc get manifestwork lab5-secrets-testing-app-272c0 -n lab5 -o yaml | yq
oc get multiclusterapplicationsetreports
```

Controllers for multiclusterapplicationsetreports
```
oc logs multicluster-integrations-* -n open-cluster-management
```
- resource sync controller
- aggregation controller
- propagation controller
