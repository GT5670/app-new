# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

Not sure about the team, but I don't like the current setup of the release notes in our repository. 
Reasons:
I have not seen release notes in the assemblies directory.
Why making it hard when we can keep it simple.
For a new joiner in our team (not saying we are going to have one... LOL) it'll be easier to understand the repository structure.

For a couple of projects related to us, here is how:
This is how RHDH handles their release notes - https://github.com/redhat-developer/red-hat-developers-documentation-rhdh/tree/main/titles
This is how Openshift handles their release notes - https://github.com/openshift/openshift-docs/tree/main/release_notes

I want the best of both the worlds.

So this is what I propose:
Removing release notes from the assemblies directory
Using the release notes that we have in title
Then follow, what Openshift does - https://github.com/openshift/openshift-docs/tree/main/release_notes (Clone the repo and you can visualize the things better)

The way I see it, this does not require significant restructuring. But if it does, I believe we can move this to 1.3, but remember, then we will have a lot of content to manage and restructure.

If you and the team are averse to this proposal we can discuss more.
