# argocd-helm-hooks-sample
Sample repo to demonstrate that helm hook delete mechanics are fixed as in https://github.com/argoproj/argo-cd/issues/4384 .

## Background
[Argo CD](https://github.com/argoproj/argo-cd) supports helm hooks where a helm hook is translated into an Argo CD resource hook and sync waves. A slightly different handling of hook weight and hook delete policies in earlier Argo CD versions could lead to resources being deleted too early / earlier than with native helm cli. This issue is described in detail in https://github.com/argoproj/argo-cd/issues/4384 and should luckily be fixed with https://github.com/argoproj/gitops-engine/pull/144.

## Sample chart
This sample repo reproduces the scenario that lead to the problem in earlier Argo CD versions with a minimum of k8s resources. The helm chart in [the chart directory](chart/) consists of 3 resources:

- **[configmap helm-hook-pre-cm](chart/templates/configmap-presync.yaml)** as hook in helm phases "pre-install,pre-upgrade" = Argo CD phase "presync", weight "-6" and delete policy "before-hook-creation, hook-succeeded"
- **[job helm-hook-pre-job](chart/templates/job-presync.yaml)** as hook in helm phases "pre-install,pre-upgrade" = Argo CD phase "presync", weight "-5" and delete policy "before-hook-creation"
    - depends on configmap helm-hook-pre-cm (uses a value as env var)
    - starts a single pod that prints the configmap value, sleeps 180 seconds and then finishes with printing "done"
- **[configmap my-synced-cm](chart/templates/configmap-sync.yaml)** as regular, "permanent" resource in the regular sync phase - this configmap is meant to stay :blush:

## Expected behavior with the PR / fix (everything's good)
We expect the following chain of events:

1. presync hook phase (helm: pre-install/pre-upgrade)
    1. hook weight / sync wave -6
        1. configmap helm-hook-pre-cm is deleted if existing (hook-delete-policy "before-hook-creation")
        2. configmap helm-hook-pre-cm is created
    2. hook weight / sync wave -5
        1. job helm-hook-pre-job is deleted if existing (hook-delete-policy "before-hook-creation")
        2. job helm-hook-pre-job is created :arrow_right: pod prints "Hello my friend!", sleeps 180 seconds and exits
    3. all hooks finished in presync hook phase
        1. configmap helm-hook-pre-cm is deleted (hook-delete-policy "hook-succeeded")
2. sync phase
    1. configmap my-synced-cm is created

I'm not entirely sure about the position of 1.2.1 (deletion of job helm-hook-pre-job because of before-hook-creation delete policy), this could also happen at the same time as 1.1.1 (this would mean that all resources in the presync phase with hook-delete-policy before-hook-creation would be deleted at the same time, i.e. at the beginning of the phase REGARDLESS of the hook-weight / sync wave).
This does not impact the outcome of this sample and is irrelevant to verify the fix in the PR mentioned above.

With Argo CD v2.0.1 I could verify the above chain of events, the sample worked as expected.

## Expected behavior without the fix / with older Argo CD versions (e.g. v1.5)
With older versions of Argo CD without the fix from the issue/PR above the following should happen:

1. presync hook phase (helm: pre-install/pre-upgrade)
    1. hook weight / sync wave -6
        1. configmap helm-hook-pre-cm is deleted if existing (hook-delete-policy "before-hook-creation")
        2. configmap helm-hook-pre-cm is created
        3. **configmap helm-hook-pre-cm is deleted (hook-delete-policy "hook-succeeded")**
    2. hook weight / sync wave -5
        1. job helm-hook-pre-job is deleted if existing (hook-delete-policy "before-hook-creation")
        2. job helm-hook-pre-job is created :arrow_right: **pod crashes** because configmap helm-hook-pre-cm does not exist (anymore)

I have not verified this exact sample with an older Argo version but we ran into exactly this problem with another helm chart in ArgoCD v1.5.

## Further information
See the following links for more technical information about helm hooks, resource hooks and sync waves in Argo CD:

- [helm hooks in Argo CD](https://argoproj.github.io/argo-cd/user-guide/helm/#helm-hooks)
- [resource hooks in Argo CD (helm hooks are translated into this)](https://argoproj.github.io/argo-cd/user-guide/resource_hooks/)
- [sync waves in Argo CD](https://argoproj.github.io/argo-cd/user-guide/sync-waves/)
- [Argo CD issue with more technical background](https://github.com/argoproj/argo-cd/issues/4384)