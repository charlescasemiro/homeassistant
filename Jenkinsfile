pipeline {
    agent none

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    environment {
        HA_DEPLOY_DIR = '/home/pokehouse/homeassistant'
        GH_REPOSITORY = 'charlescasemiro/homeassistant'
        GH_WORKFLOW = 'homeassistant-image-security-update.yaml'
    }

    stages {
        stage('Checkout master') {
            agent {
                label 'homeassistant-docker'
            }
            steps {
                checkout scm
                script {
                    env.HA_GIT_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Detect Home Assistant image update') {
            agent {
                label 'home-assistant'
            }
            steps {
                script {
                    def images = sh(
                        script: '''
                            set -eu
                            docker compose \
                              --project-directory "$HA_DEPLOY_DIR" \
                              --project-name homeassistant \
                              --file "$WORKSPACE/docker-compose.yaml" \
                              config --images
                        ''',
                        returnStdout: true
                    ).trim().readLines().findAll {
                        it.startsWith('ghcr.io/home-assistant/home-assistant:')
                    }

                    if (images.size() != 1) {
                        error("Expected one Home Assistant image in Compose, found ${images.size()}")
                    }

                    env.HA_TARGET_IMAGE = images[0]
                    env.HA_RUNNING_IMAGE = sh(
                        script: "docker inspect --type container --format='{{.Config.Image}}' homeassistant",
                        returnStdout: true
                    ).trim()
                    env.HA_DEPLOY_REQUIRED = (
                        env.HA_TARGET_IMAGE != env.HA_RUNNING_IMAGE
                    ).toString()

                    echo "Compose image: ${env.HA_TARGET_IMAGE}"
                    echo "Running image: ${env.HA_RUNNING_IMAGE}"
                    echo "Deployment required: ${env.HA_DEPLOY_REQUIRED}"
                }
            }
        }

        stage('Wait for GitHub Actions scan') {
            agent {
                label 'homeassistant-docker'
            }
            when {
                expression {
                    env.HA_DEPLOY_REQUIRED == 'true'
                }
            }
            steps {
                script {
                    timeout(time: 20, unit: 'MINUTES') {
                        while (true) {
                            def result = sh(
                                script: '''
                                    set -eu
                                    curl --fail --silent --show-error \
                                      --connect-timeout 10 --max-time 20 \
                                      -H 'Accept: application/vnd.github+json' \
                                      -H 'X-GitHub-Api-Version: 2022-11-28' \
                                      --get \
                                      --data-urlencode "head_sha=$HA_GIT_COMMIT" \
                                      --data-urlencode "event=push" \
                                      --data-urlencode "branch=master" \
                                      --data-urlencode "per_page=10" \
                                      "https://api.github.com/repos/$GH_REPOSITORY/actions/workflows/$GH_WORKFLOW/runs" |
                                    python3 -c '
import json
import os
import sys

data = json.load(sys.stdin)
runs = [
    run for run in data.get("workflow_runs", [])
    if run.get("head_sha") == os.environ["HA_GIT_COMMIT"]
    and run.get("event") == "push"
]
run = max(runs, key=lambda item: item.get("run_number", 0)) if runs else None
if run is None:
    state, url = "missing", ""
elif run.get("status") == "completed":
    state, url = run.get("conclusion") or "unknown", run.get("html_url", "")
else:
    state, url = run.get("status", "unknown"), run.get("html_url", "")
print(state)
print(url)
'
                                ''',
                                returnStdout: true
                            ).trim()

                            def fields = result.readLines()
                            def state = fields[0]
                            env.GITHUB_ACTIONS_RUN_URL = fields.size() > 1 ? fields[1] : ''

                            if (state == 'success') {
                                echo "GitHub Actions scan succeeded: ${env.GITHUB_ACTIONS_RUN_URL}"
                                break
                            }
                            if (state in ['missing', 'queued', 'requested', 'waiting', 'in_progress']) {
                                echo "GitHub Actions status is ${state}; checking again in 30 seconds."
                                sleep(time: 30, unit: 'SECONDS')
                                continue
                            }

                            error("GitHub Actions scan ended with status: ${state}")
                        }
                    }
                }
            }
        }

        stage('Approve Home Assistant deployment') {
            agent none
            when {
                expression {
                    env.HA_DEPLOY_REQUIRED == 'true'
                }
            }
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input(
                        message: "Review the Trivy report, then approve deployment of ${env.HA_TARGET_IMAGE}. GitHub Actions run: ${env.GITHUB_ACTIONS_RUN_URL}",
                        ok: 'Deploy Home Assistant'
                    )
                }
            }
        }

        stage('Pull and restart Home Assistant') {
            agent {
                label 'homeassistant-docker'
            }
            when {
                expression {
                    env.HA_DEPLOY_REQUIRED == 'true'
                }
            }
            steps {
                sh '''
                    set -eu
                    docker compose \
                      --project-directory "$HA_DEPLOY_DIR" \
                      --project-name homeassistant \
                      --file "$WORKSPACE/docker-compose.yaml" \
                      pull homeassistant
                    docker compose \
                      --project-directory "$HA_DEPLOY_DIR" \
                      --project-name homeassistant \
                      --file "$WORKSPACE/docker-compose.yaml" \
                      up -d --no-deps homeassistant
                    docker compose \
                      --project-directory "$HA_DEPLOY_DIR" \
                      --project-name homeassistant \
                      --file "$WORKSPACE/docker-compose.yaml" \
                      ps homeassistant
                '''
            }
        }
    }
}
