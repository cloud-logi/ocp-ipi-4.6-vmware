# Instalacion OCP 4.6 en Vmware Metodo IPI
### Dhcp en lo posible asigne hasta 3 nameservers para evitar no tener que modificar parametros del ocp
Crear estos registros dns
 ```
API VIP  api.<cluster_name>.<base_domain>.
Entrada VIP  *.apps.<cluster_name>.<base_domain>.
 ```
Generar clave privada
 ```
# ssh-keygen
# eval "$(ssh-agent -s)"
# ssh-add <path>/<file_name>
  ```
Bajar los programas
https://cloud.redhat.com/openshift/create/datacenter

Agregar certificados de CA raíz de vCenter a la confianza de su sistema
  ```
# wget --no-check-certificate https://vcenter/certs/download.zip
# cp certs/lin/* /etc/pki/ca-trust/source/anchors
# update-ca-trust extract
  ```

 Creando el archivo de configuración de la instalación
 ```
# ./openshift-install create install-config --dir=ocp
 ```

Archivo de inicio de config install-config.yaml

--------------------------------------------------------------------------
```
apiVersion: v1
baseDomain: logicalis.local
#Compute nodes definition
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 2
  platform:
    vsphere:
      cpus: 4
      coresPerSocket: 4
      memoryMB: 8192
      osDisk:
        diskSizeGB: 120
#Master node definition
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3
  platform:
    vsphere:
      cpus: 4
      coresPerSocket: 2
      memoryMB: 16384
      osDisk:
        diskSizeGB: 120
metadata:
  creationTimestamp: null
  name: prev
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.242.0/24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  vsphere:
    apiVIP: ...
    cluster: ...
    datacenter: ...
    defaultDatastore: ...
    ingressVIP: ...
    network: ...
    password: ..
    username: ..
    vCenter: 10.54...
    folder: "/../vm/openshift4/ocp4"
publish: External
fips: false
pullSecret: '{"auths":{"cloud.openshift.com":{"auth":" "registry.redhat.io":{"auth":"}}'
sshKey: 'ssh-rsa ..............'
```
--------------------------------------------------------------------------

Creamos los Manifest
```
# ./openshift-install create manifest --dir=ocp
```
```
# cd ocp/manifest
```

Config del archivo cluster-network-03-config-yml
--------------------------------------------------------------------------
```
apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec: 
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
  defaultNetwork:
    type: OpenShiftSDN
    openshiftSDNConfig:
      mode: NetworkPolicy
      mtu: 1450
      vxlanPort: 4789 ----->> 5789 ---->> El proximo va a 6789
```
--------------------------------------------------------------------------
Por debajo del 9000 y sube mil al que esta ahora 

Implementar el clúster
```
# ./openshift-install create cluster --dir=ocp
```

## Crear los nodos Infra 
https://docs.openshift.com/container-platform/4.6/machine_management/creating-infrastructure-machinesets.html

Extraer infraestructura id
```
# oc get -o jsonpath='{.status.infrastructureName}{"\n"}' infrastructure cluster
prev-rkh2g
```

00-machineset_infra.yaml
--------------------------------------------------------------------------
```
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  generation: 3
  labels:
    machine.openshift.io/cluster-api-cluster: prev-rkh2g
  name:  prev-rkh2g-infra
  namespace: openshift-machine-api
spec:
  replicas: 0
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: prev-rkh2g
      machine.openshift.io/cluster-api-machineset: prev-rkh2g-infra
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: prev-rkh2g
        machine.openshift.io/cluster-api-machine-role: infra
        machine.openshift.io/cluster-api-machine-type: infra
        machine.openshift.io/cluster-api-machineset: prev-rkh2g-infra
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/infra: ""
      providerSpec:
        value:
          apiVersion: vsphereprovider.openshift.io/v1beta1
          credentialsSecret:
            name: vsphere-cloud-credentials
          diskGiB: 120
          kind: VSphereMachineProviderSpec
          memoryMiB: 16384
          metadata:
            creationTimestamp: null
          network:
            devices:
            - networkName: ACI_LAB_DEMO|ACI_AppProf|EPG-HX-Openshift
          numCPUs: 8
          numCoresPerSocket: 4
          snapshot: ""
          template: prev-rkh2g-rhcos
          userDataSecret:
            name: worker-user-data
          workspace:
            datacenter: HYPERFLEX-INNO-ARG
            datastore: HX-DP-OPC-MAXI
            folder: /HYPERFLEX-INNO-ARG/vm/openshift4/ocp4
            resourcePool: /HYPERFLEX-INNO-ARG/host/HYPERFLEX-INNO-ARG/Resources
            server: 10.54.153.150
```
--------------------------------------------------------------------------
```
# oc project openshift-machine-api
```
```
# oc get machinesets -n openshift-machine-api
```
```
# oc create -f 00-machineset_infra.yaml
```
```
# oc get machinesets -n openshift-machine-api
```

Escalar el machineset 

CREAR EL MACHINE SET POOLS

Eliminar label worker
```
# oc edit nodes prev-rkh2g-infra-mwt2t
Eliminar ---->     node-role.kubernetes.io/worker: ""
```

Agregar taint no escheduler al infra Para que no resiba pods
```
# oc adm taint nodes prev-rkh2g-infra-mwt2t node-role.kubernetes.io/infra:NoSchedule
node/prev-rkh2g-infra-mwt2t tainted
```

Movemos los Ingress 
```
# oc edit ingresscontroller default -n openshift-ingress-operator
```
Agregar noseSelector y toleration en la seccion spec

```
  spec:
    nodePlacement:
      nodeSelector:
        matchLabels:
          node-role.kubernetes.io/infra: ""
      tolerations:
        - effect: NoSchedule 
          key: node-role.kubernetes.io/infra 
          operator: Exists 
```
Ahora vemos como van migrando los pods, Si no migran hay que borrarlos.
```
# oc get pods -n openshift-ingress -o wide
```
Soporte de RedHat recomendo subir las replicas del ingress a 3

Mover Registros
```
# oc edit configs.imageregistry.operator.openshift.io cluster
```
Agregar nodeSelector y toleration en la seccion de spec

```
  nodeSelector:
    node-role.kubernetes.io/infra: ""
  tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
      operator: Exists

```
Verificar que se hayan movido 

