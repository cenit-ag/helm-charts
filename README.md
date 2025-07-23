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

