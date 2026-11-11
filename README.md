# homeassistant

## Home Assistant image security updates

The `Home Assistant image security update` workflow runs weekly, on demand, and
when the workflow definition changes on `master`. It finds the latest stable
Home Assistant Core release, scans the corresponding container image with Trivy,
and opens or updates a pull request with the image version in
`docker-compose.yaml`. The full vulnerability report is retained as a workflow
artifact.

The workflow only proposes an update; it does not deploy or merge it. Review and
merge the pull request, then update the running Compose environment separately.
For pull-request creation to work, enable **Allow GitHub Actions to create and
approve pull requests** in the repository's Actions settings.
