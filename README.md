

This repository contains the documentation for openHAB.x.

The result is available at [https://openhab.org/docs/](https://www.openhab.org/docs/) and [https://openhab.org/addons/](https://www.openhab.org/addons/).

## How it works

In this repo you can find and improve all _general_ documentation contents.
In fact, that is all you can see in the `main` branch.
There are other _read-only_ branches, which hold external content like the _add-ons_ and _concepts_ documentation, which is explained in more details below.

### So I cannot improve an add-on article here?

Correct, this is done in the original repository of the add-on.
You may want to know how to find the right file in all of those repos?
This is fairly easy:
on most of the documentation pages on <https://openhab.org/>,
you will find the following link at the bottom, which will point you directly to the file you want to improve.

![Contribution link to a specific page](./images/contribution_link.png)

When your improvement has been made and merged, we will get the updated article automatically through our build mechanism.
This happens mostly once a day. Afterwards your change is included in the next build of the openHAB website.

## Contributing to the Documentation

The documentation is a community effort, so everyone is welcome to suggest changes, add new sections and fix bugs.
This is done exactly the same way as for the code repositories, simply through pull requests against this repo.
When editing a page through the _"Edit this page on GitHub"_ link on the website, you will be given the opportunity to
create a pull request directly from GitHub.
Please read our [contribution guidelines](.github/CONTRIBUTING.md) and try to follow
them as best as you can before submitting a change for review - but don't worry if you don't understand all of them, we
will help you to get it right.

## So what are the other branches for?

We use them to bring together all relevant articles or to archive versioned content.
Mostly those branches will get updated automatically through our continuous integration builds.
You can read a bit more below about our external resources and how we get them.

### Automatically Generated Parts

Those parts include **all** add-on documentation files, no matter if they are from the `openhab-core` repo, the `openhab-addons` repo or any special binding repo like _habmin_, _zwave_ or the _alexa skill_.

We are keeping all those files at their original location, because it simply doesn't make sense to keep them here.
Imagine you want to do an improvement of the zwave binding and have to update the readme file in a completely different place.
That's twice the effort and also we would have to coordinate two Pull Requests.
So we are saving time for everyone by keeping those files at their original location along with the code.

### How the documentation build works

We have set up our [build server](https://ci.openhab.org/view/Documentation/) to do the magic automatically.
There are several triggers (mostly time based), which will then _gather the external contents_ and move them to our [final](https://github.com/openhab/openhab-docs/tree/final) branch.
You can find this migrated external content in the _final_ branch under:

- `_addons_*`
- `concepts`

You can even have a look at how this works in detail.
The external content is updated by the following toolchain:

- `update-external-resources.sh` → `pom.xml` → `process_addons.groovy`

Everything that gets updated in the _master_ branch will be also merged over to the _final_ branch automatically.
Afterwards we will redeploy the website with the latest content from the _final_ branch at regular intervals.

#### Build triggers investigated

There are two triggers available currently.
The `merge docs` job is triggered after something has been added to the documentation through this repository.
The `gather external docs` job is started with a **successful** build of the openhab-distribution.
A successful distribution build will include all the latest changes that have been made to ex