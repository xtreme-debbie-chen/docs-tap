# Creating accelerators

This topic describes how to create an accelerator in Tanzu Application Platform GUI. An accelerator contains your enterprise-conformant code and configurations that developers can use to create new projects that by default follow the standards defined in your accelerators.

## <a id="creating-accelerators-prerequisites"></a>Prerequisites

The following prerequisites are required to create an accelerator:

  - Application Accelerator is installed. For information about installing Application Accelerator, see [Installing Application Accelerator for VMware Tanzu](../install-app-acc.md)
  - You can access Tanzu Application Platform GUI from a browser OR have configured the Application Accelerator extension for VS Code. 
      - For more information on the Tanzu Application Platform GUI, see the corresponding section in the most recent release of the [Tanzu Application Platform documentation](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/index.html). 
      - For more information on the Application Accelerator extension for VS Code, see the latest on the [Application Accelerator Visual Studio Code extension page](../vscode.md)
  - kubectl is installed. The Kubernetes command line tool (kubectl) is installed and authenticated with admin rights for your target cluster.

## <a id="creating-accelerators-getting-started"></a>Getting started

You can use any Git repository to create an accelerator. You need the URL of the repository to create an accelerator.

For this example, the Git repository is `public` and contains a `README.md` file. These are options available when you create repositories on GitHub.

Use the following procedure to create an accelerator based on this Git repository:

1. Clone your Git repository.

2. Create a file named `accelerator.yaml` in the root directory of this Git repository.

3. Add the following content to the `accelerator.yaml` file:

    ```yaml
    accelerator:
      displayName: Simple Accelerator
      description: Contains just a README
      iconUrl: https://images.freecreatives.com/wp-content/uploads/2015/05/smiley-559124_640.jpg
      tags:
      - simple
      - getting-started
    ```

    Feel free to use a different icon if it uses a reachable URL.

4. Add the new `accelerator.yaml` file, commit this change, and push to your Git repository.

## <a id="publishing-the-new-accelerator"></a>Publishing the new accelerator

1. To publish your new accelerator, run this command in your terminal:

    ```sh
    tanzu accelerator create simple --git-repository ${GIT_REPOSITORY_URL} --git-branch ${GIT_REPOSITORY_BRANCH}
    ```

    Where:

    - `GIT_REPOSITORY_URL` is the URL for your Git repository where the accelerator is located.
    - `GIT_REPOSITORY_BRANCH` is the name of the branch where you pushed the new `accelerator.yaml` file.

2. Refresh Tanzu Application Platform GUI or the Application Accelerator extension in VS Code to reveal the newly published accelerator.

    ![Screenshot showing the new Simple Accelerator included in Tanzu Application Platform GUI.](../images/new-accelerator-deployed-v1-1.png)

    >**Note** It might take a few seconds to refresh the catalog and add an entry for your new accelerator.

An alternative is to use the Tanzu CLI to create a separate manifest file and apply it to the cluster:

1. Create a `simple-manifest.yaml` file and add the following content, filling in with your Git repository and branch values.

    ```yaml
    apiVersion: accelerator.apps.tanzu.vmware.com/v1alpha1
    kind: Accelerator
    metadata:
      name: simple
      namespace: accelerator-system
    spec:
      git:
        url: YOUR-GIT-REPOSITORY-URL
        ref:
          branch: YOUR-GIT-BRANCH
    ```

2. To apply the `simple-manifest.yaml`, run this command in your terminal in the directory where you created this file:

    ```sh
    tanzu accelerator apply -f simple-manifest.yaml
    ```

## <a id="using-local-path"></a>Using local-path for publishing accelerators

You can publish an accelerator directly from a local directory on your system. This helps when authoring accelerators and allows you to avoid having to commit every small change to a remote Git repository.

>**Note** You can also specify `--interval` so the accelerator is reconciled quicker when we push new changes.

```sh
tanzu accelerator create simple --local-path ${ACCELERATOR_PATH} --source-image ${SOURCE_IMAGE_REPO} --interval 10s
```

Where:

- `ACCELERATOR_PATH` is the path to the accelerator source. It can be fully qualified or a relative path. If your current directory is already the directory where your source is, then use ".".
- `SOURCE_IMAGE_REPO` is the name of the OCI image repository where you want to push the new accelerator source. If using Docker Hub, use something such as `docker.io/YOUR_DOCKER_ID/simple-accelerator-source`.

After you have made any additional changes you can push the latest to the same OCI image repository using:

```sh
tanzu accelerator push --local-path ${ACCELERATOR_PATH} --source-image ${SOURCE_IMAGE_REPO}
```

The accelerator now reflects the new content after approximately a 10 second wait since we specified that as the interval when we created the accelerator above.

## <a id="using-accelerator-fragments"></a>Using accelerator fragments

Accelerator fragments are reusable accelerator components that can provide options, files or transforms. They may be imported to accelerators using an `import` entry and the transforms from the fragment may be referenced in an `InvokeFragment` transform in the accelerator that is declaring the import. For additional details see [InvokeFragment transform](transforms/invoke-fragment.md).

The accelerator samples include three fragments - `java-version`, `tap-initialize`, and `live-update`. See the [vmware-tanzu/application-accelerator-samples/fragments](https://github.com/vmware-tanzu/application-accelerator-samples/tree/tap-1.3/fragments) Git repository for the content of these fragments.

To discover what fragments are available to use, you can run the following command:

```
tanzu accelerator fragment list
```

Look a the `java-version` fragment as an example. It contains the following `accelerator.yaml` file:

```
accelerator:
  options:
  - name: javaVersion
    inputType: select
    label: Java version to use
    choices:
    - value: "1.8"
      text: Java 8
    - value: "11"
      text: Java 11
    - value: "17"
      text: Java 17
    defaultValue: "11"
    required: true

engine:
  merge:
    - include: [ "pom.xml" ]
      chain:
      - type: ReplaceText
        regex:
          pattern: "<java.version>.*<"
          with: "'<java.version>' + #javaVersion + '<'"
    - include: [ "build.gradle" ]
      chain:
      - type: ReplaceText
        regex:
          pattern: "sourceCompatibility = .*"
          with: "'sourceCompatibility = ''' + #javaVersion + ''''"
    - include: [ "config/workload.yaml" ]
      chain:
      - type: ReplaceText
        condition: "#javaVersion == '17'"
        substitutions:
          - text: "spec:"
            with: "'spec:\n  build:\n    env:\n    - name: BP_JVM_VERSION\n      value: \"17\"'"
```

This fragment contributes the following to any accelerator that imports it:

1. An option named `javaVersion` with three choices `Java 8`, `Java 11`, and `Java 17`
2. Three `ReplaceText` transforms:
    - if the accelerator has a `pom.xml` file then what is specified for `<java.version>` is replaced with the chosen version.
    - if the accelerator has a `build.gradle` file then what is specified for `sourceCompatibility` is replaced with the chosen version.
    - if the accelerator has a `config/workload.yaml` file and the user selected "Java 17" then a build environment entry of BP_JVM_VERSION is inserted into the `spec:` section.

## <a id="deploy-accelerator-frags"></a>Deploying accelerator fragments

To deploy new fragments to the accelerator system you can use the new `tanzu accelerator fragment create` CLI command or you can apply a custom resource manifest file with either `kubectl apply` or the `tanzu accelerator apply` commands.

The resource manifest for the `java-version` fragment looks like this:

```
apiVersion: accelerator.apps.tanzu.vmware.com/v1alpha1
kind: Fragment
metadata:
  name: java-version
  namespace: accelerator-system
spec:
  displayName: Select Java Version
  git:
    ref:
      tag: GIT_TAG_VERSION
    url: https://github.com/vmware-tanzu/application-accelerator-samples.git
    subPath: fragments/java-version
```
Where `GIT_TAG_VERSION` is the git tag of the `java-version` fragment. As an example, `tap-1.4.0` is a valid git tag for the `java-version` fragment.

To create the fragment (we can save the above manifest in a `java-version.yaml` file) and use:

```
tanzu accelerator apply -f ./java-version.yaml
```

>**Note** The `accelerator apply` command can be used to apply both Accelerator and Fragment resources.

To avoid having to create a separate manifest file, you can use the following command instead:

```
tanzu accelerator fragment create java-version \
  --git-repo https://github.com/vmware-tanzu/application-accelerator-samples.git \
  --git-tag ${GIT_TAG_VERSION} \
  --git-sub-path fragments/java-version
```

Where `GIT_TAG_VERSION` is the git tag of the `java-version` fragment. As an example, `tap-1.4.0` is a valid git tag for the `java-version` fragment.

Now you can use this `java-version` fragment in an accelerator:

```
accelerator:
  displayName: Hello Fragment
  description: A sample app
  tags:
  - java
  - spring
  - cloud
  - tanzu

  imports:
  - name: java-version

engine:
  merge:
    - include: ["**/*"]
    - type: InvokeFragment
      reference: java-version
```

The earlier acelerator imports the `java-version` which, as seen earlier, provides an option to select the Java version to use for the project. It then instructs the engine to invoke the transforms provided in the fragment that updates the Java version used in `pom.xml` or `build.gradle` files from the accelerator.

For more detail on the use of fragments, see [InvokeFragment transform](transforms/invoke-fragment.md).

## <a id="Next-steps"></a>Next steps

Learn how to:

- Write an [accelerator.yaml](accelerator-yaml.md).
- Configure accelerators with [Accelerator Custom Resources](accelerator-crd.md).
- Manipulate files using [Transforms](transform-intro.md).
- Use [SpEL in the accelerator.yaml file](spel-samples.md).
