# Kapp controller and flux git repo integration


This repo is a very simple example of an experiment using the `gitRepository` resource from flux as a source for kapp controller apps. Thsi is something that could eventually be natively implemented in kapp, for info and tracking on that see this [github issue](https://github.com/carvel-dev/kapp-controller/issues/1335). The idea behind this is that flux already has a re-usable resource that pull down source code and stores it in an artifact that is locally available on the cluster. Particularly when using Tanzu Mission Control a user may already be using flux and kapp and having kapp also pull the source down is redundant. It also presents the challenge of having to specify credentials for both the app resource and the flux gitrepo resources which again is redundant. So why not just re-use the existing git repository resource that already has credentials to the repo added? That is what this experiment makes possible.


## Pre-reqs

This was setup with an existing git repo that TMC created. that is not a requirement, but you will need to create a `gitRepository` resource. 

Becuase this was using TMC it also created the initial `Kustomization`, so you will need to create a `Kustomization` that points at the `base` folder in this repo.

# How it works

This makes use of the [secretGen Controller](https://github.com/carvel-dev/secretgen-controller) from Carvel, specifically the `secretTemplate` [resource](https://github.com/carvel-dev/secretgen-controller/blob/develop/docs/secret-template.md). You can find the details in the `secretGen` directory. Essentially what this does is the following:

1. the initial kustomization pointed at `base` creates the `secretTemplate`. This template looks at the `gitRepository` resource's `status.artifact.url` and creates a secret out of that value. This is the real magic here. We need to be able to reference this field from the kapp `app` and this is a major component of doing that.
2. the initial kustomization pointed at `base` creates the kapp-flux-sample `kustomization`. This kustomization uses the `postBuild` variable substitution to inject the value from the secret created in the first step. This is the artifact url.
3. the kapp-flux-sample kustomization that is in the `apps` directory create a kapp application and uses the `${artifactUrl}` as it's http source. This now allows for kapp to use the source pulled by the flux provided gitrepository resource.
4. kapp deploys a simple app from that source.



# Why it works

This works becuase the secret template allows us to reach into any k8s resources and grab fields from them and create a secret. That paired with the ability of flux to substitute variables from a secret on post build allow us to now get that artifact url and pass it downstream to the flux created objects. In this case that is the kapp app. Kapp then allows us to use an http source that is a `tar.gz` and natively handles unpacking that. Finally becuase the secretTemplate constantly reconciles it picks up new values for the artifact url and updates the secret, then when the kustomization reconciles it get the new artifact url. This means it continuously updates for us.