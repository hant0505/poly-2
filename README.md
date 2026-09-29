# Poly

```console
 ____        ___                
/\  _`\     /\_ \               
\ \ \L\ \___\//\ \    __  __    
 \ \ ,__/ __`\\ \ \  /\ \/\ \   
  \ \ \/\ \L\ \\_\ \_\ \ \_\ \  
   \ \_\ \____//\____\\/`____ \ 
    \/_/\/___/ \/____/ `/___/> \
                          /\___/
                          \/__/ 

Poly will help you deploy multi-app.
```

## Usage

Poly is used as a dependency (subchart).

### Step by step to install

Login to the registry. If you do not have permission to access the registry, see [this manual](#install-without-connecting-to-registry) to install Poly

```console
helm registry login registry.ftech.ai/is-chart
```

Create your chart, then remove boilerplates

```console
helm create mychart

# remove boilerplates, be careful the path
rm -rf mychart/templates/*
```

Add dependency in `Chart.yaml`

```yaml
...
dependencies:
  - name: poly
    repository: oci://registry.ftech.ai/is-chart
    version: "*.*.*" # this will use latest version, change it to the version you want
```

Update dependency

```console
helm dependency update
```

Add your app config in `values.yaml`

```yaml
...
poly:
  apps:
    <app-name>: # ex: nginx
      enabled: true
      # Your app config goes here
  ...
```

## Install without connecting to registry

Ask admin for poly package. It's a `tgz` file such as `poly-1.0.0.tgz`  
Create your chart, then remove boilerplates

```console
helm create mychart

# remove boilerplates, be careful the path
rm -rf mychart/templates/*
```

Put `poly-<version>.tgz` file into `charts` directory  
Add dependency in `Chart.yaml`

```yaml
...
dependencies:
  - name: poly
    repository: file://charts/poly-<version>.tgz
    version: "<version>" # ex: "1.0.0"
```

Add your app config in `values.yaml`

```yaml
...
poly:
  apps:
    <app-name>: # ex: nginx
      enabled: true
      # Your app config goes here
  ...
```

## Get Poly config format and default values

You can get app config format of Poly by running command:

```console
helm show values oci://registry.ftech.ai/is-chart/poly > poly-values.yaml
```

Or this if you do not have permission to access the registry
> Note: Replace with your Poly version

```console
helm show values charts/poly-<version>.tgz > poly-values.yaml
```

## Push to registry

After upgrade poly source code, you can push to the registry. Remember to login registry first.

Package Poly chart

```console
# In poly directory
helm package . -d release
```

Push to the registry, replace `${VERSION}` with the one you packaged.

```console
helm push release/poly-${VERSION}.tgz oci://registry.ftech.ai/is-chart
```

## Config new project

- `nameOverride` and `chart.name`: project name
- Labels for all resources, add to `apps.<app-name>.global.extraLabels`:
  - `owner: <project name> | <department name>`
  - `pic: <requester name, ex: namtt>`
- Default resource

```yaml
resources:
  limits:
    cpu: 256m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi
```

- `command` and `runtimeArgs`: ask requester
- Volumes (if any): ask requester or self-config
- Port number:
  - With `cms` usually is `1337`
  - Or you can ask requester
- Prometheus ServiceMonitor, if any, ask requester
- `health_check` ingress path: ask requester or self-config
- Ingress:
  - domain: requester will send to you
  - tls: if for `prod` environment, ask requester, otherwise leave it blank
- Vault:
  - They will send in `pastebin`
