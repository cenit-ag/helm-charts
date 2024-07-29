# CENIT Helm Charts
Charts in a Helm-compliant repository for products by CENIT or other vendors.

# Usage

Add the chart repository:
```
helm repo add cenit https://cenit-ag.github.io/helm-charts
```

Run an update to gather the up-to-date chart index to your client:
```
helm repo update
```

List all charts available in the repository (flag `-l` will show all versions available for all matching charts):
```
helm search repo -l cenit
```

Example output:
```
NAME            CHART VERSION   APP VERSION  [...]
cenit/sm        1.2.0                        [...]
cenit/sm        1.1.8                        [...]
...
cenit/sm        1.1.0                        [...]
cenit/sm        1.0.3                        [...]
cenit/test      0.1.0           1.16.0       [...]
```

Pull a chart:
```
helm pull cenit/sm --version <chart-version>
```

If an expected chart does not show up after running `helm repo update`, delete the repo, add and update it once again:
```
helm repo remove cenit
helm repo add cenit https://cenit-ag.github.io/helm-charts
helm repo update
```

If the problem persists, please [create an issue](https://github.com/cenit-ag/helm-charts/issues/new/choose).

# Maintenance

__NOTICE:__ This part is only revelant for CENIT staff maintaining this repository.

## Configure for Github Pages

A Github repository can only be used a Helm repository if the branch containing the charts is configured as a Github pages site.

* Enable the feature in "Settings > Pages".
* Select branch `main` and directory `root` for Github Pages.

Further, only public repositories can uses the Pages feature or free. 

## Adding or Updating a Chart

### Clone

Clone this repository with Git:
```
git clone https://github.com/cenit-ag/helm-charts.git
```

### Package

Package the chart (i.e. a chart named `test`) with the root directory of this repository as destination:
```
helm package test/ --version <chart-version> --destination .
```

### Docs

Copy the chart's README into the charts subfolder and name it with the chart's corresponding version number. Example for chart `test`:
```
test/docs/sm-<chart-version>.md
```

### Index

Index repository. Run this command from the repo's root directory:
```
helm repo index --url https://cenit-ag.github.io/helm-charts .
```

### Commit

Commit and push your changes to the Git repository.
