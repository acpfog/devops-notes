## How to configure gcloud

### Create a configuration

```console
$ gcloud config configurations create config-name
```

### Set parameters for the created configuration

```console
$ gcloud config set project project-name
$ gcloud config set compute/region europe-west3
$ gcloud config set compute/zone europe-west3-a
```

### Authenticate into your gcloud account

```console
$ gcloud auth login gcloud.account@example.com
```
