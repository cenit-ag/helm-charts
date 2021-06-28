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

List all charts available in the repository:
```
helm search repo cenit
```

Pull a chart:
```
helm pull cenit/test --version 0.1.0
```

# Maintenance

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
helm package test/ --destination .
```

### Index

Index repository:
```
helm repo index --url https://cenit-ag.github.io/helm-charts .
```

### Commit

Commit and push your changes to the Git repository.
