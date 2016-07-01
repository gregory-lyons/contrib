# Using SSH with git-sync

Git-sync supports using the SSH protocol for pulling Git content.

For now, only keys without password protection are supported.

## Setup

Write a config file for a Secret with name "git-creds" that holds your SSH private key, with the key mapped to "id-rsa".
```
{
  "kind": "Secret",
  "apiVersion": "v1",
  "metadata": {
    "name": "git-creds",
    "namespace": "kube-system"
  },
  "data": {
    "id-rsa": <private-key>
}
```

Create the Secret.
```
kubectl create -f /path/to/secret-config.json
```

In your Pod or Deployment configuration, specify a Secret volume to contain the "git-creds" secret.
```
volumes: [
    {
        "name": "git-secret",
        "secret": {
          "secretName": "git-creds"
        }
    },
    ...
],
```

In your git-sync container configuration, mount the Secret volume at "/etc/git-secret". Ensure that the environment variable GIT_SYNC_REPO is set to use a URL with the SSH protocol, and set GIT_SYNC_SSH to true.
```
{
    name: "git-sync",
    ...
    env: [
        {
            name: "GIT_SYNC_REPO",
            value:  "git@github.com:kubernetes/kubernetes.git",
        }, {
            name: "GIT_SYNC_SSH",
            value: "true",
        },
    ...
    ]
    volumeMounts: [
        {
            "name": "git-secret",
            "mountPath": "/etc/git-secret"
        },
        ...
    ],
}
```
**Note: Do not mount the secret volume with "readOnly: true".** Due to a peculiarity with permissions, "readOnly: true" makes the Secret readable for user, group, and others (0444), which is not restrictive enough to be used as an SSH key, yet the read-only nature prevents changing the permissions. Omitting readOnly allows for modifying the permissions from within the container.
