# Deploy an app to Dokploy

Use this runbook to deploy a Compose app or a ready-made image to Dokploy.
For Statamic with Infisical, also use [Statamic with Infisical on Dokploy](statamic-infisical-dokploy.md).
For volumes and private services, see [Dokploy volume backups](dokploy-volume-backups.md) and [Dokploy private services](dokploy-private-services.md).

## Choose a deploy model

| Model            | Use it when                                                                |
| ---------------- | -------------------------------------------------------------------------- |
| Compose from Git | Dokploy builds the image from the repository on each push                  |
| CI image         | GitHub Actions builds and tests the image, then tells Dokploy to deploy it |
| Ready-made image | You run a public image, such as a tool, without your own code              |

## Compose from Git

1. In the project, select **Create Service → Compose**.
2. Select the GitHub provider, the repository, and the branch.
3. Set the Compose path, for example `example-cms/compose.yaml`.
4. Set **Watch paths** without a leading slash, for example `example-cms/**`.
5. Open the service **Environment** tab. Add the values. Select **Save**.
6. Open **Domains**. Attach the host to the web service, for example `cms-web`, on container port `80`.
7. Turn on HTTPS and select the `letsencrypt` certificate resolver.
8. On **General**, select **Deploy**.
9. Keep **Autodeploy** on.

Expected result: the host answers over HTTPS with a valid certificate, and each push to a watched path deploys.

- A watch path with a leading slash never matches. GitHub sends paths without it. After you fix the path, redeliver the webhook in GitHub to test.
- Put values on the service **Environment** tab, not the project shared environment. Save before you deploy. A missed save gives "dependency failed to start".
- Saving the environment does not restart containers. Deploy again.
- Traefik can show its default certificate for a short time after you add a domain. Wait, then check again.

## CI image

GitHub Actions builds the image and pushes it to GHCR with the tags `:main` and `:<short-sha>`.
Then the workflow calls the Dokploy API to deploy that tag.

1. In Dokploy, turn **Autodeploy** off. Otherwise Dokploy can pull the old image before CI pushes the new one.
2. In `compose.yaml`, read the image from a variable, for example `image: ghcr.io/example/app:${APP_IMAGE_TAG}`.
3. In the workflow, set the image variable with `POST /api/compose.update`.
4. Then call `POST /api/compose.redeploy` with the `composeId`.
5. Send the key in the `x-api-key` header. Use `curl --fail-with-body`.

Rollback: set the image variable to a known good SHA tag, then redeploy.

When Autodeploy is off, the old `/api/deploy/compose/<token>` webhook fails with "Automatic deployments are disabled for this compose".
If you deploy a moving tag such as `:main`, add `pull_policy: always` to the service. Otherwise Dokploy can show "Done" and run the old local image.

## Ready-made image

1. Select **Create Service → Application**.
2. Set **Provider** to **Docker** and enter the image name.
3. Add the domain with the container port.
4. Add named volumes under **Advanced → Volumes**. Without them, each redeploy deletes the data.
5. Deploy.

## Dokploy API

- Create a key at **Settings → Profile → API/CLI**. Store it in 1Password.
- Read the OpenAPI document at `/api/settings.getOpenApiDocument`.
- Add a Compose domain with `POST /api/domain.create`:

```json
{
  "host": "cms.example.com",
  "path": "/",
  "port": 80,
  "https": true,
  "certificateType": "letsencrypt",
  "composeId": "<compose-id>",
  "serviceName": "cms-web",
  "domainType": "compose"
}
```

## Troubleshooting

| Symptom                                   | Cause and fix                                                                                                                               |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Push does not deploy                      | The watch path has a leading slash, or Autodeploy is off. Fix it, then redeliver the webhook.                                               |
| "Done" but the old version runs           | A moving tag used the local image. Add `pull_policy: always` or deploy SHA tags.                                                            |
| "dependency failed to start"              | A value is missing on the service Environment tab, or it was not saved before the deploy.                                                   |
| Build fails with `denied: denied` on GHCR | The saved registry token expired. Make a classic PAT with `write:packages`. Update **Settings → Registry**, select **Test**, then redeploy. |
| Nixpacks build fails on Node              | Nixpacks defaults to an old Node version. Set `engines.node` in `package.json`, for example `22`.                                           |
| New content in the image does not show    | A volume is mounted over that folder. Remove the mount when Git owns the content.                                                           |
