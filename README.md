# Docker OIDC Example

Here's a demo to get Docker OIDC connections working. This allows GitHub
Actions to authenticate to your Docker Hub organization and pull and push
images without you needing to manage access tokens.

This will work for both public and private GitHub / Docker repos.

## Requirements

If you want to replicate this demo on your end, you'll need:

- Docker Hub Team / Business license
- Docker Hub organization that you have access to
- Docker Hub repo created under your organization
- GitHub repo to work with

## Details

We'll wire up the OIDC connection and GitHub Actions so that when you push a
new tag to a repo on GitHub, a new Docker image will get built and pushed.
Think of this as creating a tagged release.

Given this demo only exercises using OIDC connections, the image we'll be using
to build and push is very basic. It's a custom version of Docker's hello-world
image.

## Local Testing

**Build it:**

```sh
$ docker image build --tag DOCKERHUB_ORGANIZATION_USERNAME/oidc-demo:latest .
```

**Run it:**

```sh
$ docker container run --rm DOCKERHUB_ORGANIZATION_USERNAME/oidc-demo:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## Create Docker OIDC Connection

We're going to create a connection that allows pulling and pushing from a
specific GitHub repo to a specific Docker Hub repo. It will be scoped down to
only allow this when pushing a specific tag pattern to GitHub.

You can change things around afterwards, for example maybe you want this to
happen when code lands on your main branch instead of a tag. That's no problem!
There's [examples in Docker's
documentation](https://docs.docker.com/security/authentication/oidc-connections/rulesets-claims/#subject-claims).

**Set things up on the Docker side of things:**

- Log into your Docker Hub account
- Switch to your organization
- Visit https://app.docker.com/accounts/DOCKERHUB_ORGANIZATION_NAME/admin/oidc-connections
- Create an OIDC connection:
    - Set a name that makes sense (`my-app`, `oidc-demo`, etc.)
    - Add an optional description
    - Add a rule set:
        - Name: `tagged-release` (feel free to change the name as desired)
        - Subject claim: `repo:GITHUB_USERNAME@123456/GITHUB_REPO@1234567890:ref:refs/tags/v*`
            - ^ **IMPORTANT**:
                - Adjust the above `GITHUB_` values to match your account
                - Find which IDs to use after the `@` in your GitHub repo's settings under Actions -> OIDC
        - *Technically you can add more than 1 rule set to a connection, but we won't here*
    - For the repository section:
        - Ensure "[x] Read public repositories" is checked since when we push our image we'll use `sbom: true` which requires pulling this public image  [docker/buildkit-syft-scanner](https://hub.docker.com/r/docker/buildkit-syft-scanner)
        - Repository: choose the Docker Hub repo you want this connection to apply to
        - Scope: `SCOPE-IMAGE-PUSH`
    - Click the save connection button

At this point you should have a Connection ID, it's not a secret. Copy it to
your clipboard.

## Configure GitHub Repo Variables

We'll configure (3) repo variables that the GitHub Action will reference:

- Visit https://github.com/GITHUB_USERNAME/GITHUB_REPO/settings/variables/actions
- Create a new repository variable:
    - Name it `DOCKERHUB_OIDC_CONNECTIONID` and paste your connection ID
- Create another repository variable:
    - Name it `DOCKERHUB_ORGANIZATION_USERNAME` and use your Docker Hub org name
- Create another repository variable:
    - Name it `DOCKERHUB_REPO_NAME` and use your Docker Hub org's repo name

## Configure GitHub Action

This will take care of building `amd64` and `arm64` images, then pushing it up
to your Docker Hub account when you push a new tag to GitHub:

- In your repo, create `.github/workflows/docker-publish.yml`
- Copy / paste [`docker-publish.yml` from this repo](./.github/workflows/docker-publish.yml) into yours and adjust it as desired

**One takeaway from the above file is how Docker handles logging in:**

```yml
      - name: "Docker login"
        uses: "docker/login-action@v4"
        env:
          DOCKERHUB_OIDC_CONNECTIONID: "${{ vars.DOCKERHUB_OIDC_CONNECTIONID }}"
        with:
          username: "${{ vars.DOCKERHUB_ORGANIZATION_USERNAME }}"
```

The above tells the Docker login action which Docker Hub OIDC connection to
use. GitHub provides the workflow's OIDC identity and the rule set we created
for that connection determines whether the workflow is allowed to access the
Docker Hub repository and what it can do there.

**Another takeaway is the OIDC permissions, don't forget this in your workflow:**

```yml
    permissions:
      # This is for GitHub's checkout action, normally it's set to read but
      # since we define a custom `permissions`, we'll need to set it here.
      contents: "read"
      # Allow this job to request a GitHub OIDC token, this write is not
      # associated with write access to your Docker Hub repo.
      id-token: "write"
```

**The last takeaway is how the action is run in case you want to change it:**

```yml
on:
  push:
    # This aligns with our strategy of having this run when a git tag is pushed,
    # it matches the pattern we used in the Connection ID subject claim of "v*".
    tags:
      - "v*"
```

## Push a Git Tag to Test

After everything is wired up you should be able to run:

```sh
# This steps is only needed if you're pushing code before your tag.
git push origin main

git tag v0.0.1
git push origin v0.0.1
```

At this point:

- The `docker-publish.yml` GitHub Action should kick in and run successfully
- The Docker image should be pushed to your Docker Hub organization
    - If this didn't work you can view the "Failures" tab within the Connection ID on the Docker Hub to see why it failed if it got to the point where it reached the connection

You can test it locally:

```sh
# Delete the local image we built before:
docker image rm -f DOCKERHUB_ORGANIZATION_USERNAME/oidc-demo:latest

# Run it again, which should pull from your Docker organization:
docker container run --rm DOCKERHUB_ORGANIZATION_USERNAME/oidc-demo:latest
```

Keep in mind the last command isn't using the OIDC connection. It's using
whatever Docker Hub credentials you have configured locally. This could be
facilitated through Docker Desktop or however you have Docker installed and
configured.
