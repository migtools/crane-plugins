This repository contains manifests files for different plugins containing some metadata for the plugin and location of a binary file for appropriate os/arch.

This repository should be used as an example of how a plugin-index repository for custom plugins should look like.

### Requirements:

1.  At the root of this repository there is a [index.yml](index.yml) file that contains information about where the manifest files are located. Below is an the format of the `index.yml` file:

    ```
    apiServer: crane.konveyor.io/v1alpha1
    kind: PluginIndex
    plugins:
    - name: <plugin_name>
      path : <raw-url>
    ```

    Note: the location/path (url) of the respective manifest file of a plugin should be http accessible url to make sure the file can be retrieved via a `Get` call.

2. There is [plugins](plugins) directory at the root where all the plugin-manifest files are located.  

    A plugin manifest file should be named - `<index.yaml>`.

    A plugin manifest file should be formatted in the following manner.

    ```
      apiServer: crane.konveyor.io/v1alpha1
      kind: Plugin
      versions:
      - name: <plugin-name>
        shortDescription: <short description>
        description: <description>
        version: <version>
        metadata:
        - optionalFields:
            flagName: <flag-name>
            help: <help-message>
            example: <example>
        binaries:
        - os: <os name>
          arch: <arch>
          uri: <uri of binary>
          sha: <sha of binary>
        - os: <os name>
          arch: <arch>
          uri: <uri of binary>
          sha: <sha of binary>
      - name: <plugin-name>
        shortDescription: <short description>
        description: <description>
        version: <version>
        metadata:
        - optionalFields:
            flagName: <flag-name>
            help: <help-message>
            example: <example>
        binaries:
        - os: <os name>
          arch: <arch>
          uri: <uri of binary>
          sha: <sha of binary>
        - os: <os name>
          arch: <arch>
          uri: <uri of binary>
          sha: <sha of binary>
    ```
