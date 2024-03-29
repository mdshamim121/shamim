sion: 2.1
 
orbs:
  slack: circleci/slack@2.6.0
 
references:
  shared-environment: &shared-environment
    CIRCLE_ARTIFACTS: /tmp/artifacts
    CIRCLE_TEST_REPORTS: /tmp/test_results
 
  docker-image: &docker-image
    - image: circleci/node:10.16.3-browsers
 
  setup-results-and-artifacts: &setup-results-and-artifacts
    name: Create Directories for Results and Artifacts
    command: |
      mkdir -p                                  \
        "$CIRCLE_ARTIFACTS/icfy"                \
        "$CIRCLE_ARTIFACTS/notifications-panel" \
        "$CIRCLE_ARTIFACTS/translate"           \
        "$CIRCLE_ARTIFACTS/screenshots"         \
        "$CIRCLE_ARTIFACTS/full-site-editing"   \
        "$CIRCLE_ARTIFACTS/wpcom-block-editor"  \
        "$CIRCLE_TEST_REPORTS/client"           \
        "$CIRCLE_TEST_REPORTS/eslint"           \
        "$CIRCLE_TEST_REPORTS/integration"      \
        "$CIRCLE_TEST_REPORTS/packages"         \
        "$CIRCLE_TEST_REPORTS/server"           \
        "$CIRCLE_TEST_REPORTS/e2ereports"       \
        "$HOME/jest-cache"                      \
        "$HOME/terser-cache"
 
  # Jest cache caching
  #
  # Jest uses a cache to speed up builds. If we persist this cache across builds,
  # we can improve the speed of subsequent builds.
  #
  # Circle caches never overwritten, so we must ensure that Jest caches from different jobs
  # do not collide or we'll only cache 1 job.
  #
  # We also need to ensure that different nodes and different total nodes do not collide.
  # When we split tests, different nodes will receive a different set of tests so each node's
  # cache should be unique.
  #
  # Finally, we cache on the branch and revision, falling back to master. This should give us
  # pretty good "nearest neighbor" primer for the Jest cache.
  #
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-jest-cache: &restore-jest-cache
    name: Restore Jest cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}
  save-jest-cache: &save-jest-cache
    name: Save Jest cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-{{ .Branch }}-{{ .Revision }}
    paths:
      - ~/jest-cache
 
  #
  # Terser cache caching
  #
  restore-terser-cache: &restore-terser-cache
    name: Restore Terser cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser
  save-terser-cache: &save-terser-cache
    name: Save Terser cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-{{ .Branch }}-{{ .Revision }}
    paths:
      - ~/terser-cache
 
  # Git cache
  #
  # Calypso is a big repository with a lot of history. It can take a long time to do a full checkout.
  # By including the `.git` directory in the cache, we can speed things up by only needing to update
  # the local repository.
  #
  # We cache on the branch and revision, falling back to master, or any recent cache.
  #
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-git-cache: &restore-git-cache
    name: Restore git cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git
  update-git-master: &update-git-master
    name: Update master branch
    command: git fetch --force origin master
  save-git-cache: &save-git-cache
    name: Save git cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-{{ .Branch }}-{{ .Revision }}
    paths:
      - '.git'
 
  # npm cache
  #
  # npm caches the modules it installs in ~/.npm.
  # We cache that cache to save time pulling modules from the network.
  #
  #
  # Some prior art:
  #   - https://github.com/Automattic/wp-calypso/pull/25431
  #   - https://github.com/Automattic/wp-calypso/pull/25487
  #   - https://github.com/Automattic/wp-calypso/pull/27180
  #
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-npm-cache: &restore-npm-cache
    name: 'Restore npm cache'
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v3-npm-modules-{{ checksum ".nvmrc" }}-{{ checksum "package-lock.json" }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v3-npm-modules-{{ checksum ".nvmrc" }}
 
  npm-install: &npm-install
    name: Install npm dependencies
    command: npm ci
 
  save-npm-cache: &save-npm-cache
    name: 'Save node_modules cache'
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v3-npm-modules-{{ checksum ".nvmrc" }}-{{ checksum "package-lock.json" }}
    paths:
      - ~/.npm
  npm-e2e-install: &npm-e2e-install
    name: Install e2e npm dependencies
    command: |
      cd test/e2e &&
      CHROMEDRIVER_VERSION=$(<.chromedriver_version) npm ci
 
  # Babel cache
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-babel-client-cache: &restore-babel-client-cache
    name: Restore Babel Client Cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client
  save-babel-client-cache: &save-babel-client-cache
    name: Save Babel Client Cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-{{ .Branch }}-{{ .Revision }}
    paths:
      - 'build/.babel-client-cache'
 
  set-e2e-variables: &set-e2e-variables
    name: Set e2e environment variables
    command: |
      echo 'export NODE_CONFIG_ENV=test' >> $BASH_ENV &&
      echo 'export BRANCHNAME=$CIRCLE_BRANCH' >> $BASH_ENV &&
      echo 'export HIGHLIGHT_ELEMENT=true' >> $BASH_ENV
 
  move-e2e-artifacts: &move-e2e-artifacts
    name: Move report and screenshots
    when: always
    command: |
      cp -r ./reports $CIRCLE_TEST_REPORTS/e2ereports &&
      cp -r ./screenshots $CIRCLE_ARTIFACTS/screenshots
 
  defaults: &defaults
    working_directory: ~/wp-calypso
    docker: *docker-image
    environment: *shared-environment
 
commands:
  prepare:
    description: 'Prepare Job'
    steps:
      # folders to collect results
      - run: *setup-results-and-artifacts
      - attach_workspace:
          at: '~'
  store-artifacts-and-test-results:
    description: Stores artifacts and test results
    steps:
      - store_test_results:
          path: /tmp/test_results
      - store_artifacts:
          path: /tmp/test_results
      - store_artifacts:
          path: /tmp/artifacts
 
jobs:
  setup:
    <<: *defaults
    steps:
      # repo
      - restore_cache: *restore-git-cache
      - checkout
      - run: *update-git-master
      - save_cache: *save-git-cache
      # npm dependencies
      - restore_cache: *restore-npm-cache
      - run: *npm-install
      - run: *npm-e2e-install
      - save_cache: *save-npm-cache
      - run: npm run build-packages
      - persist_to_workspace:
          root: '~'
          paths:
            - wp-calypso
 
  typecheck:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: TypeScript typecheck
          # Report but don't fail the build (`|| exit 0`)
          command: npm run typecheck || exit 0
 
 
  lint-and-translate:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Lint Config Keys
          when: always
          command: npm run lint:config-defaults
      - run:
          name: Lint Client and Server
          when: always
          command: |
            # We may not have files to lint which returns non-0 exit
            # Ensure this does not cause job failure (see `|| exit 0`)
            FILES_TO_LINT=$(
              git diff --name-only --diff-filter=d origin/master... \
                | grep -E '^(client/|server/|packages/)'            \
                | grep -E '\.[jt]sx?$'
            ) || exit 0
 
            if [[ ! -z $FILES_TO_LINT ]]; then
              ./node_modules/.bin/eslint                                \
                --format junit                                          \
                --output-file "$CIRCLE_TEST_REPORTS/eslint/results.xml" \
                $FILES_TO_LINT
            fi
      - run:
          name: Build calypso-strings.pot
          when: always
          command: |
            npm run translate
            mv calypso-strings.pot "$CIRCLE_ARTIFACTS/translate"
      - run:
          name: Build New Strings .pot
          when: always
          command: |
            git clone --single-branch --depth=1 https://github.com/Automattic/gp-localci-client.git
            bash gp-localci-client/generate-new-strings-pot.sh "$CIRCLE_BRANCH" "$CIRCLE_SHA1" "$CIRCLE_ARTIFACTS/translate"
            rm -rf gp-localci-client
      - store-artifacts-and-test-results
      - run:
          name: Notify GlotPress translations are ready
          when: always
          command: |
            curl -X POST https://translate.wordpress.com/api/localci/-relay-new-strings-to-gh \
              -H 'Cache-Control: no-cache' \
              -H 'Content-Type: application/json' \
              -d '{
                    "payload": {
                      "branch": "'"$CIRCLE_BRANCH"'",
                      "build_num": '"$CIRCLE_BUILD_NUM"',
                      "pull_requests": [
                        {
                          "url": "'"$CIRCLE_PULL_REQUEST"'"
                        }
                      ],
                      "reponame": "'"$CIRCLE_PROJECT_REPONAME"'",
                      "username": "'"$CIRCLE_PROJECT_USERNAME"'",
                      "vcs_revision": "'"$CIRCLE_SHA1"'",
                      "vcs_type": "github"
                    }
                  }'
 
  build-notifications:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build Notifications Panel
          command: |
            NODE_ENV=production npx lerna run build --scope='@automattic/notifications' -- -- --output-path=$CIRCLE_ARTIFACTS/notifications-panel
      - store-artifacts-and-test-results
 
  build-o2-blocks:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build Gutenberg Blocks for internal p2s
          command: |
            NODE_ENV=production npx lerna run build --scope='@automattic/o2-blocks' -- -- --output-path=$CIRCLE_ARTIFACTS/o2-blocks
      - store-artifacts-and-test-results
 
  build-wpcom-block-editor:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build the block editor in WordPress.com integration utils package
          command: |
            npx lerna run build --scope='@automattic/wpcom-block-editor' -- -- --output-path=$CIRCLE_ARTIFACTS/wpcom-block-editor
      - store-artifacts-and-test-results
 
  build-full-site-editing:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build the Full Site Editing plugin and theme
          command: |
            npx lerna run build --scope='@automattic/full-site-editing'
            cp -R apps/full-site-editing/full-site-editing-plugin $CIRCLE_ARTIFACTS/full-site-editing/
      - store-artifacts-and-test-results
  test-full-site-editing:
    <<: *defaults
    parallelism: 1
    working_directory: ~/wp-calypso/apps/full-site-editing
    steps:
      - prepare
      - run:
          name: Install Full Site Editing specific test deps
          command: |
            npm ci
      - run:
          name: Run Full Site Editing client tests
          command: |
            npm run test:unit
  test-client:
    <<: *defaults
    parallelism: 6
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Client Tests
          no_output_timeout: 2m
          command: |
            # Use Jest to list tests to run via config
            ./node_modules/.bin/jest              \
              --listTests                         \
              --config=test/client/jest.config.js \
              > ~/jest-tests
 
            # Run jest on the CircleCI split for parallelization across containers
            # Avoid using `--split-by=timings` here so that per-node Jest caches
            # receive a stable sub-set of tests for optimal cache usage.
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/client/results.xml" \
              ./node_modules/.bin/jest                                  \
                --cacheDirectory="$HOME/jest-cache"                     \
                --ci                                                    \
                --maxWorkers=2                                          \
                --reporters=default                                     \
                --reporters=jest-junit                                  \
                --runTestsByPath                                        \
                --silent                                                \
                --config=test/client/jest.config.js                     \
                $( circleci tests split < ~/jest-tests )
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  test-integration:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Integration Tests
          command: |
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/integration/results.xml" \
              ./node_modules/.bin/jest                     \
                --cacheDirectory="$HOME/jest-cache"        \
                --ci                                       \
                --maxWorkers=2                             \
                --reporters=default                        \
                --reporters=jest-junit                     \
                --silent                                   \
                --config=test/integration/jest.config.js
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  test-packages:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Package Tests
          no_output_timeout: 2m
          command: |
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/packages/results.xml" \
              ./node_modules/.bin/jest                                    \
                --cacheDirectory="$HOME/jest-cache"                       \
                --ci                                                      \
                --maxWorkers=2                                            \
                --reporters=default                                       \
                --reporters=jest-junit                                    \
                --silent                                                  \
                --config=test/packages/jest.config.js
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  test-server:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Server Tests
          no_output_timeout: 2m
          command: |
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/server/results.xml" \
              ./node_modules/.bin/jest                                  \
                --cacheDirectory="$HOME/jest-cache"                     \
                --ci                                                    \
                --maxWorkers=2                                          \
                --reporters=default                                     \
                --reporters=jest-junit                                  \
                --silent                                                \
                --config=test/server/jest.config.js
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  icfy-stats:
    <<: *defaults
    steps:
      - prepare
      - restore_cache: *restore-babel-client-cache
      - restore_cache: *restore-terser-cache
      - run:
          name: Build Stats
          environment:
            NODE_ENV: 'production'
            BROWSERSLIST_ENV: 'defaults'
          command: |
            #
            # This block should not cause a test failure and block PRs.
            # The shell should never error and exit 0 to indicate success.
            #
            set +o errexit
            npm run build-css                                      \
              && npm run preanalyze-bundles                        \
              && node bin/icfy-analyze.js                          \
              && mv stats.json chart.json style.json "$CIRCLE_ARTIFACTS/icfy" \
              || rm -fr build/.babel-client-cache # In case of failure do not save a potentially bad cache
            exit 0
      - save_cache: *save-terser-cache
      - save_cache: *save-babel-client-cache
      - store-artifacts-and-test-results
      - run:
          name: Notify ICFY of failed build
          when: on_fail
          command: |
            #
            # This block should not cause a test failure and block PRs.
            # The shell should never error and exit 0 to indicate success.
            #
            set +o errexit
            curl -X POST                                                                     \
              "http://api.iscalypsofastyet.com:5000/submit-stats-failed?secret=$ICFY_SECRET" \
              -H 'Cache-Control: no-cache'                                                   \
              -H 'Content-Type: application/json'                                            \
              -d '{
                    "payload": {
                      "branch": "'"$CIRCLE_BRANCH"'",
                      "build_num": '"$CIRCLE_BUILD_NUM"',
                      "sha": "'"$CIRCLE_SHA1"'"
                    }
                  }'
      - run:
          name: Notify ICFY
          command: |
            #
            # This block should not cause a test failure and block PRs.
            # The shell should never error and exit 0 to indicate success.
            #
            set +o errexit
            if [ -e "$CIRCLE_ARTIFACTS/icfy/stats.json" ] && [ -e "$CIRCLE_ARTIFACTS/icfy/chart.json" ]; then
              ANCESTOR_SHA1=$(git merge-base HEAD origin/master)
              curl -X POST                                                              \
                "http://api.iscalypsofastyet.com:5000/submit-stats?secret=$ICFY_SECRET" \
                -H 'Cache-Control: no-cache'                                            \
                -H 'Content-Type: application/json'                                     \
                -d '{
                      "payload": {
                        "branch": "'"$CIRCLE_BRANCH"'",
                        "build_num": '"$CIRCLE_BUILD_NUM"',
                        "sha": "'"$CIRCLE_SHA1"'",
                        "ancestor": "'"$ANCESTOR_SHA1"'"
                      }
                    }'
            fi
 
  # Prime calypso.live so it has a build ready
  #
  # We can send a request to calypso.live so that it gets a build ready.
  # This saves time waiting later when waiting for the calypso.live
  #
  # Expected usage:
  #   - After setup
  #   - Only on branches (!master)
  #
  prime-calypso-live:
    docker:
      - image: buildpack-deps
    working_directory: ~/wp-calypso
    steps:
      - run:
          name: Prime calypso.live
          command: |
            if [[ -z $CIRCLE_PR_USERNAME ]]; then
              curl --silent "https://hash-$CIRCLE_SHA1.calypso.live"
            fi
 
  # Wait for calypso.live to be ready
  #
  # Expected usage:
  #   - After main tests have passed (test-client, test-server)
  #   - Before e2e tests run
  #
  wait-calypso-live:
    docker:
      - image: buildpack-deps
    working_directory: ~/wp-calypso
    steps:
      - run:
          name: Check external author
          command: |
            if [[ ! -z $CIRCLE_PR_USERNAME ]]; then
              echo 'PRs from external authors cannot run on calypso.live'
              exit 1
            fi
      - prepare
      - run:
          name: Wait for calypso.live build
          command: ./test/e2e/scripts/wait-for-running-branch.sh
 sion: 2.1
 
orbs:
  slack: circleci/slack@2.6.0
 
references:
  shared-environment: &shared-environment
    CIRCLE_ARTIFACTS: /tmp/artifacts
    CIRCLE_TEST_REPORTS: /tmp/test_results
 
  docker-image: &docker-image
    - image: circleci/node:10.16.3-browsers
 
  setup-results-and-artifacts: &setup-results-and-artifacts
    name: Create Directories for Results and Artifacts
    command: |
      mkdir -p                                  \
        "$CIRCLE_ARTIFACTS/icfy"                \
        "$CIRCLE_ARTIFACTS/notifications-panel" \
        "$CIRCLE_ARTIFACTS/translate"           \
        "$CIRCLE_ARTIFACTS/screenshots"         \
        "$CIRCLE_ARTIFACTS/full-site-editing"   \
        "$CIRCLE_ARTIFACTS/wpcom-block-editor"  \
        "$CIRCLE_TEST_REPORTS/client"           \
        "$CIRCLE_TEST_REPORTS/eslint"           \
        "$CIRCLE_TEST_REPORTS/integration"      \
        "$CIRCLE_TEST_REPORTS/packages"         \
        "$CIRCLE_TEST_REPORTS/server"           \
        "$CIRCLE_TEST_REPORTS/e2ereports"       \
        "$HOME/jest-cache"                      \
        "$HOME/terser-cache"
 
  # Jest cache caching
  #
  # Jest uses a cache to speed up builds. If we persist this cache across builds,
  # we can improve the speed of subsequent builds.
  #
  # Circle caches never overwritten, so we must ensure that Jest caches from different jobs
  # do not collide or we'll only cache 1 job.
  #
  # We also need to ensure that different nodes and different total nodes do not collide.
  # When we split tests, different nodes will receive a different set of tests so each node's
  # cache should be unique.
  #
  # Finally, we cache on the branch and revision, falling back to master. This should give us
  # pretty good "nearest neighbor" primer for the Jest cache.
  #
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-jest-cache: &restore-jest-cache
    name: Restore Jest cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}
  save-jest-cache: &save-jest-cache
    name: Save Jest cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v8-jest-{{ .Environment.CIRCLE_JOB }}-{{ .Environment.CIRCLE_NODE_INDEX }}/{{ .Environment.CIRCLE_NODE_TOTAL }}-{{ .Branch }}-{{ .Revision }}
    paths:
      - ~/jest-cache
 
  #
  # Terser cache caching
  #
  restore-terser-cache: &restore-terser-cache
    name: Restore Terser cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser
  save-terser-cache: &save-terser-cache
    name: Save Terser cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v0-terser-{{ .Branch }}-{{ .Revision }}
    paths:
      - ~/terser-cache
 
  # Git cache
  #
  # Calypso is a big repository with a lot of history. It can take a long time to do a full checkout.
  # By including the `.git` directory in the cache, we can speed things up by only needing to update
  # the local repository.
  #
  # We cache on the branch and revision, falling back to master, or any recent cache.
  #
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-git-cache: &restore-git-cache
    name: Restore git cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git
  update-git-master: &update-git-master
    name: Update master branch
    command: git fetch --force origin master
  save-git-cache: &save-git-cache
    name: Save git cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v2-git-{{ .Branch }}-{{ .Revision }}
    paths:
      - '.git'
 
  # npm cache
  #
  # npm caches the modules it installs in ~/.npm.
  # We cache that cache to save time pulling modules from the network.
  #
  #
  # Some prior art:
  #   - https://github.com/Automattic/wp-calypso/pull/25431
  #   - https://github.com/Automattic/wp-calypso/pull/25487
  #   - https://github.com/Automattic/wp-calypso/pull/27180
  #
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-npm-cache: &restore-npm-cache
    name: 'Restore npm cache'
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v3-npm-modules-{{ checksum ".nvmrc" }}-{{ checksum "package-lock.json" }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v3-npm-modules-{{ checksum ".nvmrc" }}
 
  npm-install: &npm-install
    name: Install npm dependencies
    command: npm ci
 
  save-npm-cache: &save-npm-cache
    name: 'Save node_modules cache'
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v3-npm-modules-{{ checksum ".nvmrc" }}-{{ checksum "package-lock.json" }}
    paths:
      - ~/.npm
  npm-e2e-install: &npm-e2e-install
    name: Install e2e npm dependencies
    command: |
      cd test/e2e &&
      CHROMEDRIVER_VERSION=$(<.chromedriver_version) npm ci
 
  # Babel cache
  # More about the CircleCI cache: https://circleci.com/docs/2.0/caching
  restore-babel-client-cache: &restore-babel-client-cache
    name: Restore Babel Client Cache
    keys:
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-{{ .Branch }}-{{ .Revision }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-{{ .Branch }}
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-master
      - v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client
  save-babel-client-cache: &save-babel-client-cache
    name: Save Babel Client Cache
    key: v{{ .Environment.GLOBAL_CACHE_PREFIX }}-v1-babel-client-{{ .Branch }}-{{ .Revision }}
    paths:
      - 'build/.babel-client-cache'
 
  set-e2e-variables: &set-e2e-variables
    name: Set e2e environment variables
    command: |
      echo 'export NODE_CONFIG_ENV=test' >> $BASH_ENV &&
      echo 'export BRANCHNAME=$CIRCLE_BRANCH' >> $BASH_ENV &&
      echo 'export HIGHLIGHT_ELEMENT=true' >> $BASH_ENV
 
  move-e2e-artifacts: &move-e2e-artifacts
    name: Move report and screenshots
    when: always
    command: |
      cp -r ./reports $CIRCLE_TEST_REPORTS/e2ereports &&
      cp -r ./screenshots $CIRCLE_ARTIFACTS/screenshots
 
  defaults: &defaults
    working_directory: ~/wp-calypso
    docker: *docker-image
    environment: *shared-environment
 
commands:
  prepare:
    description: 'Prepare Job'
    steps:
      # folders to collect results
      - run: *setup-results-and-artifacts
      - attach_workspace:
          at: '~'
  store-artifacts-and-test-results:
    description: Stores artifacts and test results
    steps:
      - store_test_results:
          path: /tmp/test_results
      - store_artifacts:
          path: /tmp/test_results
      - store_artifacts:
          path: /tmp/artifacts
 
jobs:
  setup:
    <<: *defaults
    steps:
      # repo
      - restore_cache: *restore-git-cache
      - checkout
      - run: *update-git-master
      - save_cache: *save-git-cache
      # npm dependencies
      - restore_cache: *restore-npm-cache
      - run: *npm-install
      - run: *npm-e2e-install
      - save_cache: *save-npm-cache
      - run: npm run build-packages
      - persist_to_workspace:
          root: '~'
          paths:
            - wp-calypso
 
  typecheck:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: TypeScript typecheck
          # Report but don't fail the build (`|| exit 0`)
          command: npm run typecheck || exit 0
 
 
  lint-and-translate:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Lint Config Keys
          when: always
          command: npm run lint:config-defaults
      - run:
          name: Lint Client and Server
          when: always
          command: |
            # We may not have files to lint which returns non-0 exit
            # Ensure this does not cause job failure (see `|| exit 0`)
            FILES_TO_LINT=$(
              git diff --name-only --diff-filter=d origin/master... \
                | grep -E '^(client/|server/|packages/)'            \
                | grep -E '\.[jt]sx?$'
            ) || exit 0
 
            if [[ ! -z $FILES_TO_LINT ]]; then
              ./node_modules/.bin/eslint                                \
                --format junit                                          \
                --output-file "$CIRCLE_TEST_REPORTS/eslint/results.xml" \
                $FILES_TO_LINT
            fi
      - run:
          name: Build calypso-strings.pot
          when: always
          command: |
            npm run translate
            mv calypso-strings.pot "$CIRCLE_ARTIFACTS/translate"
      - run:
          name: Build New Strings .pot
          when: always
          command: |
            git clone --single-branch --depth=1 https://github.com/Automattic/gp-localci-client.git
            bash gp-localci-client/generate-new-strings-pot.sh "$CIRCLE_BRANCH" "$CIRCLE_SHA1" "$CIRCLE_ARTIFACTS/translate"
            rm -rf gp-localci-client
      - store-artifacts-and-test-results
      - run:
          name: Notify GlotPress translations are ready
          when: always
          command: |
            curl -X POST https://translate.wordpress.com/api/localci/-relay-new-strings-to-gh \
              -H 'Cache-Control: no-cache' \
              -H 'Content-Type: application/json' \
              -d '{
                    "payload": {
                      "branch": "'"$CIRCLE_BRANCH"'",
                      "build_num": '"$CIRCLE_BUILD_NUM"',
                      "pull_requests": [
                        {
                          "url": "'"$CIRCLE_PULL_REQUEST"'"
                        }
                      ],
                      "reponame": "'"$CIRCLE_PROJECT_REPONAME"'",
                      "username": "'"$CIRCLE_PROJECT_USERNAME"'",
                      "vcs_revision": "'"$CIRCLE_SHA1"'",
                      "vcs_type": "github"
                    }
                  }'
 
  build-notifications:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build Notifications Panel
          command: |
            NODE_ENV=production npx lerna run build --scope='@automattic/notifications' -- -- --output-path=$CIRCLE_ARTIFACTS/notifications-panel
      - store-artifacts-and-test-results
 
  build-o2-blocks:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build Gutenberg Blocks for internal p2s
          command: |
            NODE_ENV=production npx lerna run build --scope='@automattic/o2-blocks' -- -- --output-path=$CIRCLE_ARTIFACTS/o2-blocks
      - store-artifacts-and-test-results
 
  build-wpcom-block-editor:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build the block editor in WordPress.com integration utils package
          command: |
            npx lerna run build --scope='@automattic/wpcom-block-editor' -- -- --output-path=$CIRCLE_ARTIFACTS/wpcom-block-editor
      - store-artifacts-and-test-results
 
  build-full-site-editing:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - run:
          name: Build the Full Site Editing plugin and theme
          command: |
            npx lerna run build --scope='@automattic/full-site-editing'
            cp -R apps/full-site-editing/full-site-editing-plugin $CIRCLE_ARTIFACTS/full-site-editing/
      - store-artifacts-and-test-results
  test-full-site-editing:
    <<: *defaults
    parallelism: 1
    working_directory: ~/wp-calypso/apps/full-site-editing
    steps:
      - prepare
      - run:
          name: Install Full Site Editing specific test deps
          command: |
            npm ci
      - run:
          name: Run Full Site Editing client tests
          command: |
            npm run test:unit
  test-client:
    <<: *defaults
    parallelism: 6
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Client Tests
          no_output_timeout: 2m
          command: |
            # Use Jest to list tests to run via config
            ./node_modules/.bin/jest              \
              --listTests                         \
              --config=test/client/jest.config.js \
              > ~/jest-tests
 
            # Run jest on the CircleCI split for parallelization across containers
            # Avoid using `--split-by=timings` here so that per-node Jest caches
            # receive a stable sub-set of tests for optimal cache usage.
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/client/results.xml" \
              ./node_modules/.bin/jest                                  \
                --cacheDirectory="$HOME/jest-cache"                     \
                --ci                                                    \
                --maxWorkers=2                                          \
                --reporters=default                                     \
                --reporters=jest-junit                                  \
                --runTestsByPath                                        \
                --silent                                                \
                --config=test/client/jest.config.js                     \
                $( circleci tests split < ~/jest-tests )
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  test-integration:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Integration Tests
          command: |
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/integration/results.xml" \
              ./node_modules/.bin/jest                     \
                --cacheDirectory="$HOME/jest-cache"        \
                --ci                                       \
                --maxWorkers=2                             \
                --reporters=default                        \
                --reporters=jest-junit                     \
                --silent                                   \
                --config=test/integration/jest.config.js
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  test-packages:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Package Tests
          no_output_timeout: 2m
          command: |
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/packages/results.xml" \
              ./node_modules/.bin/jest                                    \
                --cacheDirectory="$HOME/jest-cache"                       \
                --ci                                                      \
                --maxWorkers=2                                            \
                --reporters=default                                       \
                --reporters=jest-junit                                    \
                --silent                                                  \
                --config=test/packages/jest.config.js
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  test-server:
    <<: *defaults
    parallelism: 1
    steps:
      - prepare
      - restore_cache: *restore-jest-cache
      - run:
          name: Run Server Tests
          no_output_timeout: 2m
          command: |
            JEST_JUNIT_OUTPUT="$CIRCLE_TEST_REPORTS/server/results.xml" \
              ./node_modules/.bin/jest                                  \
                --cacheDirectory="$HOME/jest-cache"                     \
                --ci                                                    \
                --maxWorkers=2                                          \
                --reporters=default                                     \
                --reporters=jest-junit                                  \
                --silent                                                \
                --config=test/server/jest.config.js
      - save_cache: *save-jest-cache
      - store-artifacts-and-test-results
 
  icfy-stats:
    <<: *defaults
    steps:
      - prepare
      - restore_cache: *restore-babel-client-cache
      - restore_cache: *restore-terser-cache
      - run:
          name: Build Stats
          environment:
            NODE_ENV: 'production'
            BROWSERSLIST_ENV: 'defaults'
          command: |
            #
            # This block should not cause a test failure and block PRs.
            # The shell should never error and exit 0 to indicate success.
            #
            set +o errexit
            npm run build-css                                      \
              && npm run preanalyze-bundles                        \
              && node bin/icfy-analyze.js                          \
              && mv stats.json chart.json style.json "$CIRCLE_ARTIFACTS/icfy" \
              || rm -fr build/.babel-client-cache # In case of failure do not save a potentially bad cache
            exit 0
      - save_cache: *save-terser-cache
      - save_cache: *save-babel-client-cache
      - store-artifacts-and-test-results
      - run:
          name: Notify ICFY of failed build
          when: on_fail
          command: |
            #
            # This block should not cause a test failure and block PRs.
            # The shell should never error and exit 0 to indicate success.
            #
            set +o errexit
            curl -X POST                                                                     \
              "http://api.iscalypsofastyet.com:5000/submit-stats-failed?secret=$ICFY_SECRET" \
              -H 'Cache-Control: no-cache'                                                   \
              -H 'Content-Type: application/json'                                            \
              -d '{
                    "payload": {
                      "branch": "'"$CIRCLE_BRANCH"'",
                      "build_num": '"$CIRCLE_BUILD_NUM"',
                      "sha": "'"$CIRCLE_SHA1"'"
                    }
                  }'
      - run:
          name: Notify ICFY
          command: |
            #
            # This block should not cause a test failure and block PRs.
            # The shell should never error and exit 0 to indicate success.
            #
            set +o errexit
            if [ -e "$CIRCLE_ARTIFACTS/icfy/stats.json" ] && [ -e "$CIRCLE_ARTIFACTS/icfy/chart.json" ]; then
              ANCESTOR_SHA1=$(git merge-base HEAD origin/master)
              curl -X POST                                                              \
                "http://api.iscalypsofastyet.com:5000/submit-stats?secret=$ICFY_SECRET" \
                -H 'Cache-Control: no-cache'                                            \
                -H 'Content-Type: application/json'                                     \
                -d '{
                      "payload": {
                        "branch": "'"$CIRCLE_BRANCH"'",
                        "build_num": '"$CIRCLE_BUILD_NUM"',
                        "sha": "'"$CIRCLE_SHA1"'",
                        "ancestor": "'"$ANCESTOR_SHA1"'"
                      }
                    }'
            fi
 
  # Prime calypso.live so it has a build ready
  #
  # We can send a request to calypso.live so that it gets a build ready.
  # This saves time waiting later when waiting for the calypso.live
  #
  # Expected usage:
  #   - After setup
  #   - Only on branches (!master)
  #
  prime-calypso-live:
    docker:
      - image: buildpack-deps
    working_directory: ~/wp-calypso
    steps:
      - run:
          name: Prime calypso.live
          command: |
            if [[ -z $CIRCLE_PR_USERNAME ]]; then
              curl --silent "https://hash-$CIRCLE_SHA1.calypso.live"
            fi
 
  # Wait for calypso.live to be ready
  #
  # Expected usage:
  #   - After main tests have passed (test-client, test-server)
  #   - Before e2e tests run
  #
  wait-calypso-live:
    docker:
      - image: buildpack-deps
    working_directory: ~/wp-calypso
    steps:
      - run:
          name: Check external author
          command: |
            if [[ ! -z $CIRCLE_PR_USERNAME ]]; then
              echo 'PRs from external authors cannot run on calypso.live'
              exit 1
            fi
      - prepare
      - run:
          name: Wait for calypso.live build
          command: ./test/e2e/scripts/wait-for-running-branch.sh
