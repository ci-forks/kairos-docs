# QA log: kairos-io/kairos-docs#681 (re-QA after the previous FAIL)

PR: fix(gce): render the image name the pipeline publishes
Fixes: kairos-io/kairos#3644
Previous QA result: FAIL (2026-09-10) -- the version half was fixed, the flavor
half was still hardcoded `ubuntu-24-04`, so the rendered `--image` still named
an image the pipeline never publishes.
Head under test: e575590 ("fix(gce): render the image name the pipeline publishes")
merged into kairos-docs main 096e2c0. Clean merge.

## Environment

kairos-docs main   096e2c0
PR head            e575590 (ci-forks/kairos-docs, fix/gce-google-version)
merge commit        91c887a (local, clean)
node               v22.23.1, npm 10.9.8
kairos             master 96ef1c9
AuroraBoot repo    main d48afc5
auroraboot image   quay.io/kairos/auroraboot:v0.27.1 (the tag the pipeline pins)
browser            Chromium 1243 (playwright build), headless, CDP
host               Ubuntu 24.04, Linux 6.8.0-139-generic, docker 29.6.2

## 1. Clean merge into current main

    $ git merge-base HEAD pr681
    d8a6ff9cb0fb21718f9e5d4cf210379525156ce9
    $ git log --oneline d8a6ff9..096e2c0
    096e2c0 docs(kairos-factory): fix skip-step flag name
    53a8d4d Apply suggestion from @jimmykarily
    e77ed51 docs(examples): document how to set Kubernetes node labels
    20ea161 docs(cloud-config): document shebang support in stage commands
    $ git merge --no-edit pr681
    10 files changed, 171 insertions(+), 19 deletions(-)

Four commits landed on main since the PR was rebased. The merge is clean and
none of them touch the files this PR changes.

## 2. Unit tests

    $ npm run test:scripts
    # tests 6   # pass 6   # fail 0
    $ npm run test:components
    # tests 26  # pass 26  # fail 0

The GoogleImage-specific specs:

    ok 6  - renderTemplate replaces {{< GoogleImage >}} with the image name the pipeline publishes
    ok 7  - renderTemplate leaves no shortcode behind for GoogleImage
    ok 8  - renderTemplate tracks the hadron version of each docs version for GoogleImage
    ok 9  - renderTemplate ignores the flavor selector for GoogleImage
    ok 10 - renderTemplate leaves GoogleImage untouched for a docs version predating hadron
    ok 11 - renderTemplate renders GoogleImage and KairosVersion differently in one template
    ok 12 - renderTemplate substitutes every GoogleImage occurrence, not just the first
    ok 18 - buildGoogleImageName matches what upload-image-to-gcp.sh creates
    ok 19 - buildGoogleImageName leaves no dots for GCE to reject
    ok 20 - buildGoogleImageName never adds a k3s segment

## 3. Mutation check: do those specs bind to the bug?

Each mutant is applied to the merged tree, tests run, then reverted.

    baseline                                                  pass 26  fail 0
    mutant 1: drop the .replaceAll('.', '-') sanitizer         pass 17  fail 9
    mutant 2: revert flavor to hardcoded ubuntu / 24.04        pass 17  fail 9
    mutant 3: `hadronFlavorRelease === null` -> `false`        pass 25  fail 1
    mutant 4: variant 'core' -> 'standard'                     pass 17  fail 9
    restored                                                   pass 26  fail 0
    $ git diff --stat        # (empty -- tree restored)

Every part of the fix is covered, including the null guard and the deliberate
decision to fix the variant rather than read it from the flavor selector.

## 4. Built both trees

    $ git checkout 096e2c0 && npm run build      # exit 0  -> build-main
    $ git checkout qa-merge && npm run build     # exit 0  -> build-pr

## 5. Rendered `--image`, SSR HTML

    build-main (before):
    docs/installation/gce          kairos-ubuntu-24-04-core-amd64-generic-{{<
    docs/v4.3.0/installation/gce   kairos-ubuntu-24-04-core-amd64-generic-{{<
    docs/v4.2.0/installation/gce   kairos-ubuntu-24-04-core-amd64-generic-{{<
    docs/v4.1.2/installation/gce   kairos-ubuntu-24-04-core-amd64-generic-{{<

    build-pr (after):
    docs/installation/gce          kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0
    docs/v4.3.0/installation/gce   kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0
    docs/v4.2.0/installation/gce   kairos-hadron-v0-5-1-core-amd64-generic-v4-2-0
    docs/v4.1.2/installation/gce   kairos-hadron-v0-4-0-core-amd64-generic-v4-1-2

No leftover shortcode anywhere:

    $ grep -rl --include='*.html' -E 'GoogleVersion|GoogleImage' build-pr/ | wc -l
    0
    $ grep -rl --include='*.html' -E 'GoogleVersion|GoogleImage' build-main/ | wc -l
    4

(The JS-bundle hits for `GoogleImage` in build-pr are the substitution code's own
regex, which is expected.)

## 6. Rendered `--image`, hydrated DOM in a real browser

Both builds served locally, each page loaded in headless Chromium over CDP,
read out of `document.body.innerText` after hydration:

    PR:
    docs/installation/gce          ["kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0"]
    docs/v4.3.0/installation/gce   ["kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0"]
    docs/v4.2.0/installation/gce   ["kairos-hadron-v0-5-1-core-amd64-generic-v4-2-0"]
    docs/v4.1.2/installation/gce   ["kairos-hadron-v0-4-0-core-amd64-generic-v4-1-2"]

    MAIN:
    all four                       ["kairos-ubuntu-24-04-core-amd64-generic-{{<"]

The hydrated DOM agrees with SSR, so this is not an SSR-only fix.

## 7. The specific scenario my previous FAIL worried about

My FAIL said "a reader switching flavor must not be handed the name of an image
that was never created". Only Hadron is offered now, so I simulated a returning
reader whose stored pick is the retired Ubuntu flavor: set
`localStorage['selectedDistro:<version>'] = 'ubuntu;ubuntu;24.04'`, reload, read
the rendered name.

    docs/installation/gce    stored={"selectedDistro:current":"ubuntu;ubuntu;24.04"}
                             -> kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0
    docs/v4.3.0/...          stored={...:"ubuntu;ubuntu;24.04"}
                             -> kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0
    docs/v4.2.0/...          stored={...:"ubuntu;ubuntu;24.04"}
                             -> kairos-hadron-v0-5-1-core-amd64-generic-v4-2-0
    docs/v4.1.2/...          stored={...:"ubuntu;ubuntu;24.04"}
                             -> kairos-hadron-v0-4-0-core-amd64-generic-v4-1-2

Unchanged. The selector cannot produce a name for an unpublished image.

Direct call on the render module, covering the null branch that no current docs
version reaches:

    hadron v0.5.1          --image=.../kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0
    hadron null            --image=.../{{< GoogleImage >}}
    ubuntu selector, v0.4.0 --image=.../kairos-hadron-v0-4-0-core-amd64-generic-v4-3-0

The null case leaves the shortcode visible rather than inventing a name, as the
PR describes.

## 8. Regression check across the whole site

Raw HTML diffing is noise (bundle hashes change every build), so I compared the
tag-stripped rendered text of every built page.

    pages in main: 549   pages in PR: 549
    only in main: none
    only in PR  : none
    pages whose rendered TEXT differs: 4
      * docs/installation/gce/index.html
      * docs/v4.1.2/installation/gce/index.html
      * docs/v4.2.0/installation/gce/index.html
      * docs/v4.3.0/installation/gce/index.html

Exactly the 4 intended pages. This is the check that matters, since the PR
changes `renderTemplate`'s signature and that function feeds every shortcode on
the site.

## 9. Does the rendered name match what the pipeline actually publishes?

This is the half that failed last time, so I verified it four independent ways.

### 9a. The hadron container each release resolves to

Ran the pipeline's own resolver:

    $ .github/public-cloud/resolve-hadron-container-image.sh v4.3.0
    Resolved hadron version for v4.3.0: v0.5.1
    quay.io/kairos/hadron:v0.5.1-core-amd64-generic-v4.3.0
    $ .github/public-cloud/resolve-hadron-container-image.sh v4.2.0
    Resolved hadron version for v4.2.0: v0.5.1
    quay.io/kairos/hadron:v0.5.1-core-amd64-generic-v4.2.0
    $ .github/public-cloud/resolve-hadron-container-image.sh v4.1.2
    Resolved hadron version for v4.1.2: v0.4.0
    quay.io/kairos/hadron:v0.4.0-core-amd64-generic-v4.1.2

Matches `hadronFlavorRelease` in docusaurus.config.ts for all three versions,
including the v4.1.2 -> v0.4.0 exception.

### 9b. The name segments read out of the published images

Not inferred from the tag, read from `/etc/kairos-release` inside each image:

    quay.io/kairos/hadron:v0.5.1-core-amd64-generic-v4.3.0
      KAIROS_ARCH="amd64" KAIROS_FLAVOR="hadron" KAIROS_FLAVOR_RELEASE="v0.5.1"
      KAIROS_MODEL="generic" KAIROS_VARIANT="core" KAIROS_VERSION="v4.3.0"
    quay.io/kairos/hadron:v0.5.1-core-amd64-generic-v4.2.0
      KAIROS_ARCH="amd64" KAIROS_FLAVOR="hadron" KAIROS_FLAVOR_RELEASE="v0.5.1"
      KAIROS_MODEL="generic" KAIROS_VARIANT="core" KAIROS_VERSION="v4.2.0"
    quay.io/kairos/hadron:v0.4.0-core-amd64-generic-v4.1.2
      KAIROS_ARCH="amd64" KAIROS_FLAVOR="hadron" KAIROS_FLAVOR_RELEASE="v0.4.0"
      KAIROS_MODEL="generic" KAIROS_VARIANT="core" KAIROS_VERSION="v4.1.2"

`VARIANT=core` for all three, so `NameFromRootfs` takes the else branch and adds
no k3s segment, which is what `buildGoogleImageName` assumes.

Chaining those through AuroraBoot's `NameFromRootfs` format, the workflow's
`tar -czvf "${file%.*}.tar.gz"`, and `upload-image-to-gcp.sh`'s `sanitizeString`:

    docs/installation/gce        pipeline=kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0 docs=... MATCH
    docs/v4.3.0/installation/gce pipeline=kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0 docs=... MATCH
    docs/v4.2.0/installation/gce pipeline=kairos-hadron-v0-5-1-core-amd64-generic-v4-2-0 docs=... MATCH
    docs/v4.1.2/installation/gce pipeline=kairos-hadron-v0-4-0-core-amd64-generic-v4-1-2 docs=... MATCH

### 9c. Ran the real generator instead of trusting my transcription of it

The above reproduces the chain by hand, so I also ran AuroraBoot itself with the
exact flags `upload-cloud-images.yaml` uses, against the oldest mapping (v4.1.2
-> hadron v0.4.0, the one that would break if the per-version lookup were wrong):

    $ docker run --rm -v /var/run/docker.sock:/var/run/docker.sock --net host \
        --privileged -v "$PWD":/aurora quay.io/kairos/auroraboot:v0.27.1 --debug \
        --set "disable_http_server=true" \
        --set "container_image=docker:quay.io/kairos/hadron:v0.4.0-core-amd64-generic-v4.1.2" \
        --set "disable_netboot=true" --set "disk.bios=true" \
        --set "disk.state_size=6000" --set "state_dir=/aurora"
    exit: 0
    INF Assembled final disk image target=/aurora/kairos-hadron-v0.4.0-core-amd64-generic-v4.1.2.raw

    $ ls -la *.raw
    -rwxrwxrwx 1 root root 1452277760 kairos-hadron-v0.4.0-core-amd64-generic-v4.1.2.raw

Then the pipeline's own packaging and naming steps on that real artifact:

    $ file=$(ls *.raw); cp "$file" disk.raw
    $ tar --format=oldgnu -czf "${file%.*}.tar.gz" disk.raw
    -rw-r--r-- 1 mudler mudler 230194312 kairos-hadron-v0.4.0-core-amd64-generic-v4.1.2.tar.gz
    $ sanitizeString "$(ls *.tar.gz)"      # verbatim from upload-image-to-gcp.sh

    GCE image name the pipeline would create:  kairos-hadron-v0-4-0-core-amd64-generic-v4-1-2
    docs/v4.1.2 page renders:                  kairos-hadron-v0-4-0-core-amd64-generic-v4-1-2
    => IDENTICAL

### 9d. The names that actually exist in palette-kairos

The strongest check: what the release run really created. From the
`workflow_run` upload-cloud-images run for the v4.3.0 release
(run 34199228248, 2026-09-08):

    Resolved hadron version for v4.3.0: v0.5.1
    INF Assembled final disk image target=/aurora/kairos-hadron-v0.5.1-core-amd64-generic-v4.3.0.raw
    + tar --format=oldgnu -czvf kairos-hadron-v0.5.1-core-amd64-generic-v4.3.0.tar.gz disk.raw
    + .github/public-cloud/upload-image-to-gcp.sh kairos-hadron-v0.5.1-core-amd64-generic-v4.3.0.tar.gz v4.3.0
    + name=kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0
    Creating image 'kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0' from the raw disk tarball in GCS.
    + gcloud compute images create kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0 \
        --project=palette-kairos \
        --source-uri=gs://kairos-cloud-images/kairos-hadron-v0.5.1-core-amd64-generic-v4.3.0.tar.gz \
        --family=kairos --labels=version=v4-3-0
    + .../test-gcp-image.sh kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0
    Image test passed successfully. Proceeding with making the image public...
    Image 'kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0' is now public.

So the name the docs now render is not just correctly derived, it is the name of
an image that was created, boot-tested by `test-gcp-image.sh`, and made public.

The same run enumerated the other images already in the project:

    kairos-hadron-v0-2-0-core-amd64-generic-v4-1-0
    kairos-hadron-v0-3-0-core-amd64-generic-v4-1-1
    kairos-hadron-v0-4-0-core-amd64-generic-v4-1-2   <- docs/v4.1.2 renders this
    kairos-hadron-v0-5-1-core-amd64-generic-v4-2-0   <- docs/v4.2.0 renders this
    kairos-hadron-v0-5-1-core-amd64-generic-v4-3-0   <- docs/ and docs/v4.3.0 render this

And today's scheduled run (35046375359, 2026-09-16 02:01) confirms they are
still live rather than cleaned up:

    Looking among versions: v4-1-0 v4-1-1 v4-1-2 v4-2-0 v4-3-0
    Image for v4.3.0 is already pushed and 'force' wasn't true. Exiting.

All three names the docs render are backed by an image present in
palette-kairos as of today.

## 10. Will it go stale on the next release?

`{{< GoogleImage >}}` reads `hadronFlavorRelease`, so I checked that the release
automation maintains it rather than leaving it to a manual bump:

    scripts/release-automation.sh:311  HADRON_VERSION="$hadron_version" \
    scripts/update-docusaurus-config.mjs:23   const hadronVersion = process.env.HADRON_VERSION;
    scripts/update-docusaurus-config.mjs:128  const targetHadronRelease = coalesce(hadronVersion, template.hadronFlavorRelease);

`hadron_version` comes out of the release's component-versions JSON, so the new
shortcode tracks releases the same way the ISO-name validator already does.

## 11. The removed `only-flavors` fence class

The PR drops `{class="only-flavors=Ubuntu+24.04"}` from the gcloud fence and
says nothing consumes it. Confirmed:

    $ grep -rn "only-flavors" --include='*.ts' --include='*.tsx' --include='*.js' \
        --include='*.mjs' --include='*.css' . | grep -v node_modules
    (no output)

36 other `only-flavors` occurrences remain in docs, untouched, consistent with
the regression diff showing only 4 pages changed.

## Result

PASS. The flavor half that failed last time is fixed, verified against the real
published artifacts and the real GCE image names, and the whole-site regression
diff is clean.

## Not verified, and why

I could not run `gcloud compute instances create` itself: that needs credentials
for the `palette-kairos` project, which this QA environment does not have. The
substitute is the release run's own `test-gcp-image.sh` step, which boots the
image in GCE and gates publication on it, plus today's run confirming the image
is still present. The remaining gap is only whether GCE would reject the name for
some reason unrelated to its spelling, and the release run already created an
image under exactly that name.

## Side observation, pre-existing and not caused by this PR

`src/components/OnlyFlavors.tsx` returns `children` on both branches of its
check, so an `only-flavors` gate does nothing anywhere on the site. I flagged
this in the previous QA run too. It is why the Ubuntu-gated gcloud block was
rendering under the default Hadron flavor in the first place. Still separate,
still worth a look.
