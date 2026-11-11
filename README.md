# homeassistant

## Home Assistant image security updates

The `Home Assistant image security update` workflow runs weekly, on demand, and
when the workflow definition, Compose file, or `Jenkinsfile` changes on
`master`. It finds the latest stable Home Assistant Core release, scans the
corresponding container image with Trivy, and opens or updates a pull request
with the image version in `docker-compose.yaml`. The full vulnerability report
is retained as a workflow artifact.

The workflow only proposes an update; it does not deploy or merge it. For
pull-request creation to work, enable **Allow GitHub Actions to create and
approve pull requests** in the repository's Actions settings.

## Post-merge deployment with Jenkins

The root `Jenkinsfile` polls `master` every five minutes. When the Compose image
differs from the running Home Assistant container, it waits up to 20 minutes for
the GitHub scan on that exact commit to succeed, then pauses for a Jenkins
approval. After approval it pulls and recreates only the `homeassistant`
service, using the checked-out Compose file with `/home/pokehouse/homeassistant`
as the project directory so bind mounts continue to point at the live data.
It then verifies the running container image matches the pinned Compose image.

Create a Jenkins Pipeline job using this repository, branch `*/master`, and
script path `Jenkinsfile`. Label an executor on the Home Assistant Docker host
`home-assistant`; it needs Docker Compose v2, `curl`, Python 3, access to
the deployment directory, and Docker permissions. Jenkins reads the public
GitHub Actions run API, so no GitHub token is required. Run the job once after
setup; later master changes are polled automatically. Review the Trivy report
linked in the Jenkins approval prompt before approving deployment.
