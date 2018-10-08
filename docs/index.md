# Prerequisites
You need a few things installed

* direnv
* [BOSH additional dependencies](https://bosh.io/docs/cli-v2-install/#additional-dependencies)

# Using BUCC (BOSH, UAA, CredHub, and ConcourseCI)
For this intro, we will use [BUCC](https://github.com/starkandwayne/bucc) to set up a director easily.

We won't be using UAA, CredHub, or ConcourseCI in this introduction though.

Clone the BUCC repository, and allow direnv to run in here
```bash
git clone https://github.com/starkandwayne/bucc.git && cd bucc/
direnv allow
```

We also need the staticsite BOSH release for the demo
```bash
git clone https://github.com/shreddedbacon/staticsite-boshrelease.git
```
Clone it into the BUCC directory, so we can retain the BUCC/BOSH environment that is already configured

# Reference Material
* [BOSH Terminology](https://bosh.io/docs/terminology/)
* [BOSH Guides](https://bosh.io/docs/update-cloud-config/)
* [Ultimate Guide to BOSH](https://ultimateguidetobosh.com/)
* [BOSH, UAA, CredHub, ConcourseCI](https://github.com/starkandwayne/bucc)
* [Staticsite BOSH Release](https://github.com/shreddedbacon/staticsite-boshrelease)
