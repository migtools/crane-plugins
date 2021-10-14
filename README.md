This repository contains manifests files for different plugins containing some metadata for the plugin and location of a binary file for appropriate os/arch.

This repository should be used as an example of how a plugin-index repository for custom plugins should look like.

### Requirements:

1.  At the root of this repository there is a [index.yml](index.yml) file that contains information about where the manifest files are located. Below is an the format of the `index.yml` file:

    ```
    <plugin_name-version> : <raw-url>
    <plugin_name-version> : <raw-url>
    ```

    Note: the location (url) of the respective manifest file of a plugin should be http accessible url to make sure the file can be retrieved via a `Get` call.

2. There is [plugins](plugins) directory at the root where all the plugin-manifest files are located.  

    A plugin manifest file should be named - `<plugin_name-version.yml>`.

    A plugin manifest file should be formatted in the following manner.

    ```
        name: <plugin-name>
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

    Here is the one example for the `plugin-maniffest` - [openshift-1.0.7.yaml](plugins/openshift-1.0.7.yml)

    ```
      name: openshift
      shortDescription: openshift
      description: this is openshift
      version: 1.0.7
      binaries:
      - os: linux
        arch: amd64
        uri: https://github.com/JaydipGabani/crane-lib/releases/download/v1.0.7/openshift-v1.0.7
    ```