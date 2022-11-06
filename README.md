# Apply Kubernetes Manifests in Backstage

This custom scaffolder action allows you to do a server-side apply for any
Kubernetes manifest.

I wrote this for a [KubeCon talk](https://github.com/muvaf/cloud-native-heroku/)
without knowing much TypeScript. Be careful if you decide to use it.

## Getting Started

Add to your Backstage app.
```bash
# From your Backstage root directory
yarn add --cwd packages/backend @muvaf/kubernetes-apply
```
```bash
# To be able to keep using the built-in actions.
yarn add --cwd packages/backend @backstage/integration
```

**IMPORTANT NOTE**: Make sure the `ServiceAccount` of your Backstage app has the
necessary permissions to create the resources you want to create.

Append it to your existing actions in `packages/backend/src/plugins/scaffolder.ts`
```typescript
import { CatalogClient } from '@backstage/catalog-client';
import { createRouter, createBuiltinActions } from '@backstage/plugin-scaffolder-backend';
import { ScmIntegrations } from '@backstage/integration';
import { Router } from 'express';
import type { PluginEnvironment } from '../types';
import { kubernetesApply } from "@muvaf/kubernetes-apply";

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  const catalogClient = new CatalogClient({ discoveryApi: env.discovery });
  const integrations = ScmIntegrations.fromConfig(env.config);

  const builtInActions = createBuiltinActions({
    integrations,
    catalogClient,
    config: env.config,
    reader: env.reader,
  });

  const actions = [
      ...builtInActions,
      kubernetesApply()
  ]

  return await createRouter({
    actions,
    catalogClient,
    logger: env.logger,
    config: env.config,
    database: env.database,
    reader: env.reader,
    identity: env.identity,
  });
}
```

Done! You can now use the action in your software templates.
```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: hello-world-on-kubernetes
  title: Hello World on Kubernetes
spec:
  steps:
    - id: create-argocd-application
      name: Create ArgoCD Application
      action: kubernetes:apply
      input:
        namespaced: true
        manifest: |
          apiVersion: argoproj.io/v1alpha1
          kind: Application
          metadata:
            name: deploy
            namespace: ${{ parameters.serviceName }}
          spec:
            destination:
              namespace: ${{ parameters.serviceName }}
              server: https://kubernetes.default.svc
            project: default
            source:
              chart: ${{ (parameters.repoUrl | parseRepoUrl).repo }}
              repoURL: ghcr.io/${{ (parameters.repoUrl | parseRepoUrl).owner }}
              targetRevision: 9.9.9
            syncPolicy:
              automated:
                selfHeal: true
              syncOptions:
                - CreateNamespace=true
```