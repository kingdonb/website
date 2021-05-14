---
title: Jenkins + Flux
linkTitle: Jenkins + Flux
description: "How to use Jenkins CI for building images together with Flux CD's image update automation."
weight: 50
---

This guide explains how to configure Flux with Jenkins, with the core ideas of [GitOps Principles] in mind. Let Jenkins handle CI (or Continuous Integration: image build and test, tagging and pushing), and let Flux handle CD (or Continuous Deployment) by making use of the Image Update Automation feature.

## Declarative Artifacts

In traditional CI/CD systems like Jenkins, the CI infra is often made directly responsible for continuous delivery. Flux treats this arrangement as dangerous, and mitigates this risk by prescribing encapsulation of resources into declarative artifacts, and a strict boundary separating CI from CD.

Jenkins (or any CI system generally speaking) can be cumbersome, and CI builds are an imperative operation that can succeed or fail including sometimes for surprising reasons. Flux obviates any deploy-time link between Jenkins and production deployments, to fulfill the promises of increased reliability and repeatability with GitOps.

### Git Manifests

Flux requires YAML manifests to be kept in a git repository or similar artifact hosting, (such as bucket storage which can also be configured to maintain a revision history.) Each revision represents a new "desired state" for the cluster workloads.

Flux represents this as a [`GitRepository` custom resource](/docs/components/source/gitrepositories/).

Flux's Source Controller periodically reconciles the config repository where the cluster's YAML manifests are maintained, pulling the latest version of the named ref. To invoke the continuous deployment functionality we use a `ref` which may be updated, such as a branch or tag. Flux deploys updates from whatever commit is at the ref, every `spec.interval`.

A revision can be any commit at the head of the branch or tag, or a specific commit hash described in the field `spec.ref` in the `GitRepository`. We could also specify a semver expression here, so that Flux infers the latest tag within a range specified. There are many possible configurations.

A specific commit hash can also be listed in the `GitRepositoryRef`, though this is less common than pointing to a branch, as in this case Flux is no longer performing continuous delivery of changes, but instead reconciles with some specific fixed revision as listed and curated by hand. This behavior, though not particularly ergonomic, could be useful during an [incident response](/docs/guides/image-update/#incident-management).

Pinning to a particular revision allows an operator to handle updates explicitly and manually but still via GitOps. While this effectively disables some automated features of Flux, it is also a capable way to operate. Jenkins jobs could also write commit hashes into manifests, and while cats can live with dogs, this is not important for this example.

When adopting Flux's approach to GitOps, Jenkins should not `kubectl apply` your manifests anymore. Jenkins does not need direct access to any production clusters. The responsibilities of Jenkins are reduced from a traditional CI/CD system which does both, to only CI. The CD job is delegated entirely to Flux. From the perspective of a production cluster, Jenkins' responsibility has ended once an image artifact is built and tested, after it has been tagged for deployment and pushed to the Image Repository.

### Image Repository

Jenkins is responsible for building and testing OCI images. Those images can be tested internally on Jenkins as part of the build, before pushing, or once deployed in a non-production cluster or other staging context.

Jenkins workflows are often as varied and complex as a snowflake. Finer points of Jenkins image building are not in scope for this guide, but links and some examples are provided to show a strategy that can be used on Jenkins with Flux.

If you're getting started and maybe have not much experience with Jenkins, (or if perhaps it was the boss who said to use Jenkins,) we hope this information comes in handy.

#### Jenkins Builds OCI Images

While many companies and people may use Jenkins for building Docker images, it isn't actually true that Jenkins builds Docker images. Docker builds OCI images, and Jenkins can shell out to Docker to build images. Jenkins always uses some other tool to build OCI images. Docker may still be the most commonly reported such tool at the time of this writing, but other tools are growing in popularity like [Porter](https://porter.sh), [Buildpacks.io](https://buildpacks.io), [Earthfile](https://earthly.dev), and surely many others we have not seen.

One might use [declarative pipelines](https://www.jenkins.io/doc/book/pipeline/docker/) to include the CI configuration in application repos by writing a Jenkinsfile with many varied approaches. We might use the [docker plugin](https://plugins.jenkins.io/docker-plugin/), or a privileged pod that uses a HostPath volume to get access to `/var/run/docker.sock`.

Proposing these ideas may also land us on the naughty list, (or hopefully attract the attention of InfoSec advocates at a design review,) as these are generally acknowledged as **extremely dangerous**, and you should not ever do these things without understanding why, with untrusted code from an unverified source, or anywhere with access to a production cluster.

(That's fine, and we already acknowledged that Jenkins-CI is to be completely separated from production, according to Flux...)

We might also be running Jenkins on a Kubernetes cluster that doesn't even have Docker underneath. In this case you may want to use another tool to produce OCI images with Jenkins.

Here, we will use a privileged pod with access to Docker on the host to its most positive effect, by building the image on a single node cluster which is specially earmarked and set aside for Jenkins builds. This is so that local storage can be invoked while building and testing images. When the next stage of the pipeline is executed after the image is built, there is no need to push or pull from a remote image registry before reaching the release stage.

Now while this strategy is efficient, it will only work if Kubernetes is running Docker underneath (and that is certainly getting less common now, though it should fortunately remain possible into the future now that [Mirantis has taken over support of dockershim](https://www.mirantis.com/blog/mirantis-to-take-over-support-of-kubernetes-dockershim-2/).)

Affected Jenkins users will have to find a way of making this work, depending on your situation. None of this is in scope for Flux to explain, as Flux separates the responsibilities of CI and CD across these well-defined boundaries. Whatever tool you use for building images, this document aims to explain and show compatible choices that work well with Flux.

It should be clear that we can use any competing tool for building images, (with or without Jenkins involvement,) and much of the same advice for working with Flux will still apply.

#### What Should We Do?

We recommend users implement SemVer and/or [Sortable image tags][Sortable image tags], to enable the use of [Image Update Policies][image update guide]. It is possible for Jenkins to drive deployments with Flux in this way, without having or needing any direct access or explicit coordination with production environment or staging clusters.

GitOps principles suggest that we should manage production workloads as purely declarative artifacts that accurately describe the cluster state in enough detail to reproduce, including version information. Extrapolating the principles, we can also prescribe updates to container images with an automated process, and appropriately constrain this process to only new SemVer releases within a specified range.

Many parts are needed for a complete continuous delivery pipeline with Jenkins and Flux. References are provided below to help highlight ideas that are likely present in a functioning Jenkins image build pipeline. We'll also provide an example you may use to build images for development and testing. Then, using another Jenkins stage, we run some tests in the image that was built, and finally make some CI decisions about whether to publish/release the image for deployment.

We can do testing without an image registry or pushing or pulling any image, because Jenkins builds the image locally on the build node, with a privileged pod that has direct access to Docker on the host. So (if we're using a single-node cluster for Jenkins, or by some other tricks perhaps ...) the image is created and used with the single node's Docker daemon.

Those are some assumptions that you may need to check on... anyway, then we tag and push an image any time a new release version was tagged in Git, only after seeing the tests pass.

Jenkins provides examples of declarative pipelines that use [credentials](https://github.com/jenkinsci/pipeline-examples/blob/master/declarative-examples/simple-examples/credentialsUsernamePassword.groovy) and show how you can use string data elements collected or composed in earlier stages, to drive downstream stages or scripts in different ways, or simply [populating environment variables](https://github.com/jenkinsci/pipeline-examples/blob/master/declarative-examples/simple-examples/scriptVariableAssignment.groovy).

Another example executes certain workflow scripts [only against a particular branch](https://github.com/jenkinsci/pipeline-examples/blob/master/declarative-examples/simple-examples/whenBranchMaster.groovy). We may need to do all of these things, or similar ideas, in order to test and release new versions of our app in production.

#### Example Jenkinsfile

Adapt this if needed, or add this to a project repository with a `Dockerfile` in its root, as a file `Jenkinsfile`, and configure a [Multibranch Pipeline][Creating a Multibranch Pipeline] to trigger when new commits are pushed to any branch or tag.

Find this example in context at [kingdonb/jenkins-example-workflow](https://github.com/kingdonb/jenkins-example-workflow) where it is connected with a Jenkins server, and configured to build and push images to [docker.io/kingdonb/jenkins-example-workflow](https://hub.docker.com/r/kingdonb/jenkins-example-workflow/tags?page=1&ordering=last_updated).

```groovy
dockerRepoHost = 'docker.io'
dockerRepoUser = 'kingdonb' // (Username must match the value in jenkinsDockerSecret)
dockerRepoProj = 'jenkins-example-workflow'

// these refer to a Jenkins secret "id", which can be in Jenkins global scope:
jenkinsDockerSecret = 'docker-registry-account'

// blank values that are filled in by pipeline steps below:
gitCommit = ''
branchName = ''
unixTime = ''
developmentTag = ''
releaseTag = ''

pipeline {
  agent {
    kubernetes { yamlFile "jenkins/docker-pod.yaml" }
  }
  stages {
    // Build a Docker image and keep it locally for now
    stage('Build') {
      steps {
        container('docker') {
          script {
            gitCommit = env.GIT_COMMIT.substring(0,8)
            branchName = env.BRANCH_NAME
            unixTime = (new Date().time / 1000) as Integer
            developmentTag = "${branchName}-${gitCommit}-${unixTime}"
            developmentImage = "${dockerRepoUser}/${dockerRepoProj}:${developmentTag}"
          }
          sh "docker build -t ${developmentImage} ./"
        }
      }
    }
    // Push the image to development environment, and run tests in parallel
    stage('Dev') {
      parallel {
        stage('Push Development Tag') {
          when {
            not {
              buildingTag()
            }
          }
          steps {
            withCredentials([[$class: 'UsernamePasswordMultiBinding',
              credentialsId: jenkinsDockerSecret,
              usernameVariable: 'DOCKER_REPO_USER',
              passwordVariable: 'DOCKER_REPO_PASSWORD']]) {
              container('docker') {
                sh """\
                  docker login -u \$DOCKER_REPO_USER -p \$DOCKER_REPO_PASSWORD
                  docker push ${developmentImage}
                """.stripIndent()
              }
            }
          }
        }
        // Start a second agent to create a pod with the newly built image
        stage('Test') {
          agent {
            kubernetes {
              yaml """\
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: test
                    image: ${developmentImage}
                    imagePullPolicy: Never
                    securityContext:
                      runAsUser: 1000
                    command:
                    - cat
                    resources:
                      requests:
                        memory: 256Mi
                        cpu: 50m
                      limits:
                        memory: 1Gi
                        cpu: 1200m
                    tty: true
                """.stripIndent()
            }
          }
          options { skipDefaultCheckout(true) }
          steps {
            // Run the tests in the new test container
            container('test') {
              sh (script: "/app/jenkins/run-tests.sh")
            }
          }
        }
      }
    }
    stage('Push Release Tag') {
      when {
        buildingTag()
      }
      steps {
        script {
          releaseTag = env.TAG_NAME
          releaseImage = "${dockerRepoUser}/${dockerRepoProj}:${releaseTag}"
        }
        container('docker') {
          withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: jenkinsDockerSecret,
            usernameVariable: 'DOCKER_REPO_USER',
            passwordVariable: 'DOCKER_REPO_PASSWORD']]) {
            sh """\
              docker login -u \$DOCKER_REPO_USER -p \$DOCKER_REPO_PASSWORD
              docker tag ${developmentImage} ${releaseImage}
              docker push ${releaseImage}
            """.stripIndent()
          }
        }
      }
    }
  }
}
```

The example above should do the necessary work for development and production. When you push a commit, it will be built and tested locally on the Jenkins node, and pushed in parallel to the image repository with a [Sortable image tag][Sortable image tags]. This can be deployed automatically by Flux, with a relaxed policy for development environments. There is no requirement that the tests pass in order to deploy the latest image in development. You can add such a requirement at a later stage, if it makes sense in your use case.

The corresponding Flux feature is covered in the [using an ImagePolicy object](/docs/guides/sortable-image-tags/#using-in-an-imagepolicy-object) section of the aforementioned guide.

When you push a git tag, different workflow is used since git tags can be used for [Automating image updates to Git](/docs/guides/image-update/) in production. Using SemVer tags, you can automatically promote new tags to production via policy. In this guide we assume you will manually create and push Git tags. These builds will run `docker build` again for the tag, which will hopefully hit the cache, and will run tests again (even if already tested); the build job then pushes a new matching image tag upon successful execution of the tests.

You can find this example demonstrated in practice with a real (private) Jenkins instance at [kingdonb/example-jenkins-workflow](https://github.com/kingdonb/example-jenkins-workflow), where it has been configured to run as a Multi-branch pipeline as explained already.

Configuration of webhooks would be a logical next step if not already configured and working, so that Jenkins can trigger builds without polling. Demonstrating this CI feature is again something that is out of scope for this guide.

### Wrap Up

Your CI workflow can be based on these examples, or may turn out completely different. Jenkins has a rich ecosystem of plugins, and the Jenkinsfile is as diverse and powerful as any programming language. If you are using Jenkins already, you probably already know exactly how you want it to build images.

If you are concerned about running Docker and Kubernetes together, or if you need to use these workflows in a cluster that cannot run container images as root, or in privileged mode, for an alternative build strategy that can still work with Jenkins in rootless mode, we recommend you check out [Kubernetes examples for Buildkit](https://github.com/moby/buildkit/tree/master/examples/kubernetes) and the [Buildkit CLI for Kubectl](https://github.com/vmware-tanzu/buildkit-cli-for-kubectl). These were recently presented together at [KubeCon/CloudNativeCon EU 2021](https://www.youtube.com/watch?v=vTh6jkW_xtI).

The finer points of building OCI images in Jenkins are out of scope for this guide. These examples are meant to be kept simple (though they should be complete), and we refrain from sharing strong opinions about how CI should work here, because it's simply out of scope for Flux to weigh in about these things. We meant to show some ways that Jenkins (or any similar functioning CI tool) can interact with Flux. This works without Jenkins being connected to production clusters, only building images, and interfacing with Flux only by publishing image tags, so there is no strong coupling or inter-dependency between CI and CD.

Update deployments via Flux's [ImagePolicy](https://fluxcd.io/docs/guides/sortable-image-tags/#using-in-an-imagepolicy-object) CRD, and the Image Update Automation API.

By pushing image tags, Jenkins can update the cluster using Flux's pull-based model for updating. When a new image is pushed that has a higher-sorted image tag, and meets the range or filter specifications of the policy, Flux is made aware of it by Image Reflector API, which captures the new candidate tag as an Image resource.

[GitOps Principles]: https://www.gitops.tech/#how-does-gitops-work
[image update guide]: /guides/image-update/
[any old app (Jenkins edition)]: https://github.com/kingdonb/any_old_app/tree/jenkins
[Flux bootstrap guide]: /get-started/
[String Substitution with sed -i]: #string-substitution-with-sed-i
[Docker Build and Tag with Version]: #docker-build-and-tag-with-version
[Jsonnet for YAML Document Rehydration]: #jsonnet-for-yaml-document-rehydration
[Commit Across Repositories Workflow]: #commit-across-repositories-workflow
[01-manifest-generate.yaml]: https://github.com/kingdonb/any_old_app/blob/main/.github/workflows/01-manifest-generate.yaml
[some guidance has changed since Flux v1]: https://github.com/fluxcd/flux2/discussions/802#discussioncomment-320189
[Sortable image tags]: /guides/sortable-image-tags/
[Creating a Multibranch Pipeline]: https://www.jenkins.io/doc/book/pipeline/multibranch/#creating-a-multibranch-pipeline
[Image Automation Controllers]: /components/image/controller/
[Example of a build process with timestamp tagging]: /guides/sortable-image-tags/#example-of-a-build-process-with-timestamp-tagging
