Since setting up my GitHub workflows as detailed **[HERE](../githubworkflow)**, I have noticed that the code coverage report generation tool
that was being used was generating a lot of false negatives and certain code (mainly .except calls) were causing a "Not covered" status to be
reported.

Tarpaulin is an alternative tool that has better integration with Rust, however the downside of it is that due to the instrumentation required
it's only available on x86_64 processors and only on the Linux OS.

Luckily however, GitHub workflows allow running on exactly that architecture. This is a very simple change to make, and uploading the codecov
report to CodeCov is still the same, the only difference is how the report has been generated.

The new workflow is as follows:

```yaml
name:                           Test Coverage

on:                             [push]
jobs:
  test:
    name:                       coverage
    runs-on:                    ubuntu-latest
    container:
      image:                    xd009642/tarpaulin:develop-nightly
      options:                  --security-opt seccomp=unconfined
    steps:
      - name:                   Checkout repository
        uses:                   actions/checkout@v2

      - name:                   Generate code coverage
        run: |
          cargo +nightly tarpaulin --verbose --all-features --workspace --lib --timeout 120 --out Xml
      - name:                   Upload to codecov.io
        uses:                   codecov/codecov-action@v2
        with:
          # token:                ${{secrets.CODECOV_TOKEN}} # not required for public repos
          fail_ci_if_error:     true
```

The main difference to previous is that we use a container image for running Tarpaulin on nightly, on a specific snapshot, and
that to generate the code coverage, we use the one liner:

```
cargo +nightly tarpaulin --verbose --all-features --workspace --lib --timeout 120 --out Xml
```

Once the report is output to the XML file, the upload to Codecov remains the same!

The result is that the code coverage report seems to be a lot more reliable, and also the setup and running of
Tarpaulin is so much easier too.

For more information, Tarpaulin can be found **[HERE](https://crates.io/crates/cargo-tarpaulin)**