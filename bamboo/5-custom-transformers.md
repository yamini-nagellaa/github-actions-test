# Use custom transformers to customize GitHub Actions Importer's behavior

In this lab, you will build upon the `dry-run` command to override GitHub Actions Importer's default behavior and customize the converted workflow using "custom transformers." Custom transformers can be used to:

1. Convert items that are not automatically converted.
2. Convert items that were automatically converted using different actions.
3. Convert environment variable values differently.
4. Convert references to runners to use a different runner name in GitHub Actions.

## Prerequisites

1. Followed the [steps to set up your GitHub Codespaces](./readme.md#configure-your-codespace) environment.
2. Completed the [configure lab](./1-configure.md#configuring-credentials).
3. Completed the [dry-run lab](./4-dry-run.md).

## Perform a dry run

You will be performing a `dry-run` command to inspect the workflow that is converted by default. Run the following command within the codespace terminal:

```bash
gh actions-importer dry-run bamboo build --output-dir tmp/custom-dry-run --plan-slug DEMO-MAR --source-file-path bamboo/bootstrap/source_files/custom/bamboo.yml
```

The converted workflow generated by the above command can be seen below:

<details>
  <summary><em>Converted workflow 👇</em></summary>

```yaml
name: demo/sample_plan
on:
#   # The shortest interval you can run scheduled workflows is once every 5 minutes.
#   period: '180'
jobs:
  Custom-Dry-Run-Build:
    runs-on:
      - self-hosted
      - bamboo-runner
    steps:
    - uses: actions/checkout@v3.5.0
      with:
        clean: false
    - run: |-
        mkdir -p falcon/red
        echo wings > falcon/red/wings
        sleep 1
        echo 'Built it'
#     # This item has no matching transformer
#     - any-task/plugin-key/com.atlassian.bamboo.plugins.builder.unknowncache:
#         any-task:
#           plugin-key: com.atlassian.bamboo.plugins.builder.unknowncache
#           configuration:
#             key: cache_key
#             path: falcon/red/wings
    - uses: actions/upload-artifact@v3.1.1
      with:
        name: FOO-BAR_Red rocket built
        path: falcon/red/wings
        if-no-files-found: error
```

</details>

_Note_: You can refer to the previous [lab](./4-dry-run.md) to learn about the fundamentals of the `dry-run` command.

## Custom transformers for an unknown step

The converted workflow contains a `any-task/plugin-key/com.atlassian.bamboo.plugins.builder.unknowncache` step that was not automatically converted.  Let's write a custom transformer to handle this unknown task!

Let's answer the following questions before proceeding to write a custom transformer.

1) What is the "identifier" of the step to customize? This should be the identifier from the comment in the workflow without the version, or in other words the name untransformed step.
  - __any-task/plugin-key/com.atlassian.bamboo.plugins.builder.unknown__

2) What is the desired Actions syntax to use instead?
  - Upon conducting some research, you've discovered that the [Actions Cache](https://github.com/marketplace/actions/cache) action in the marketplace offer comparable functionality.

  ```yaml
  - uses: actions/cache@v3
    with:
      path: <path>
      key: <key>
  ```

  Now you can begin to write the custom transformer. Custom transformers use a DSL built on top of Ruby and should be defined in a file with the `.rb` file extension. You can create this file by running the following command in your codespace terminal:

  ```bash
  touch transformers.rb && code transformers.rb
  ```

  Next, you will define a `transform` method for the `any-task/plugin-key/com.atlassian.bamboo.plugins.builder.unknowncache` identifier by adding the following code to `transformers.rb`:

  ```ruby
  transform "any-task/plugin-key/com.atlassian.bamboo.plugins.builder.unknowncache" do |item|
    [
      {
        "uses" => "actions/cache@v3",
        "with" => {
          "path" => item["configuration"]["path"],
          "key" => item["configuration"]["key"]
        }
      }
    ]
  end
  ```

This method can use any valid Ruby syntax and should return a `Hash` or `Array` that represents the YAML that should be generated for a given step. GitHub Actions Importer will use this method to convert a step with the provided identifier and will use the `item` parameter for the original values configured in Bamboo. The Bamboo task configuration can be accessed in the `item` parameter via `item["configuration"]["<item_configuration_key>"]`.

```yaml
plugin-key: com.atlassian.bamboo.plugins.builder.unknowncache
configuration:
  key: cache_key
  path: falcon/red/wings
```

To access the values of the  `key` and `path` information in the transformer, we are using `item["configuration"]["key"]` and `item["configuration"]["path"]`.

Now you can perform another `dry-run` command and use the `--custom-transformers` CLI option to provide this custom transformer. Run the following command within your codespace terminal:

```bash
gh actions-importer dry-run bamboo build --output-dir tmp/custom-dry-run --plan-slug DEMO-MAR --source-file-path bamboo/bootstrap/source_files/custom/bamboo.yml --custom-transformers transformers.rb
```

The converted workflow that is generated by the above command will now use the custom logic for the `unknowncache` step.

## Custom transformers for runners

Next, we will use a custom transformers to dictate which runners the converted workflows should use. To do this, answer the following questions:

1. What is the label of the runner in Bamboo to change?
    - __bamboo-runner__

2. What is the label of the runner in GitHub Actions to use instead?
    - __some-other-runner__

With these questions answered, you can add the following code to the `transformers.rb` file:

```ruby
runner "bamboo-runner", "some-other-runner"
```

In this example, the first parameter to the `runner` method is the runner label in Bamboo and the second is the new runner label to use in GitHub Actions.

Now you can perform another `dry-run` command with the `--custom-transformers` CLI option. When you open the converted workflow the `runs-on` statement will use the customized runner label:

```bash
gh actions-importer dry-run bamboo build --output-dir tmp/custom-dry-run --plan-slug DEMO-MAR --source-file-path bamboo/bootstrap/source_files/custom/bamboo.yml --custom-transformers transformers.rb
```
> Note: we are using a different pipeline than before that defines a `runs-on` tag with the target label.

```diff
runs-on:
-  - bamboo-runner
+  - some-other-runner
```

At this point the file contents of `transformers.rb` should match this:

<details>
  <summary><em>Custom transformers 👇</em></summary>

```ruby
runner "bamboo-runner", "some-other-runner"

transform "any-task/plugin-key/com.atlassian.bamboo.plugins.builder.unknowncache" do |item|
  [
    {
      "uses" => "actions/cache@v3",
      "with" => {
        "path" => item["configuration"]["path"],
        "key" => item["configuration"]["key"]
      }
    }
  ]
end
```

</details>

Congratulations! You have overridden GitHub Actions Importer's default behavior by customizing the conversion of:

- Unknown steps
- Runners

## Next lab
[Perform a production migration of a Bamboo pipeline](6-migrate.md)
