---
title: "Leveraging KRM Functions"
date: 2022-08-11T10:34:55-04:00
draft: false
authors:
- Ankur Mundra
tags:
- KRM Function
- Crane
---
This post discusses a possible solution to handle the execution of KRM functions and a way to leverage them.

KRM Functions
=============
The Kubernetes Resource Model (KRM) functions are small, interoperable, and language-independent executable programs packaged as containers that can be chained together as part of a configuration management pipeline and perform crud operation on Kubernetes resources.

<p>
  <img src="https://user-images.githubusercontent.com/20452032/184165434-c81e08f8-9141-42a6-8c95-51b435f2ff4f.svg">
  <img src="https://user-images.githubusercontent.com/20452032/184165446-c451ee0d-230c-4e78-a1bc-d7548d3f5c9a.svg">
</p>

where:
- `input items`: The input list of KRM resources to operate on.
- `output items`: The output list obtained from adding, removing, or modifying items in the input.
- `functionConfig`: An optional meta resource containing the arguments to this invocation of the function.
- `results`: An optional meta resource emitted by the function for observability and debugging purposes.

Leveraging KRM Functions
========================
Now, we know What KRM functions are? How valuable they can be? But how do we leverage these functions? To leverage the benefits of KRM functions we need some way to handle function execution.

1. ***Crane Subcommand***:

    A subcommand in the crane command line interface that can run KRM functions with the functionconfig.
    
    ### Command
    Crane subcommand will follow this synopsis:
    ```
    crane runfn [IMAGE] [flags] [-- args]
    ```
    > The above subcommand will be used for transforming resources in a directory using functions. If a function fails, the process is aborted and the         resource files will be left unchanged.

    #### Args:
      *IMAGE:*
    >	Container image of the function to execute e.g. quay.io/krm-fn/set-annotation:v3.1.

      *args:*
    >	Arguments to pass to the function. The value can be in `key=value` format and come after the separator **'--'**

      *Flags:*
    >

        --export-dir, e:
          Path to the local directory containing resources. 
          Defaults to the export directory.

        --transform-dir, t:
          If specified, the output resources are written to provided location,
          if not specified, resources are written to transform directory.

        --env:
          List of local environment variables to be exported to the container function.
          The value can be in key=value format or only the key of an already exported environment variable.

    #### Examples:

    ```
    # apply function example-fn on the resources in export directory and write output back to DIR
    $ crane runfn quay.io/example.com/example-fn --export-dir export
    ```

    ```
    # apply function example-fn with an input ConfigMap containing `data: {foo: bar}`
    $ crane runfn gcr.io/example.com/example-fn -- foo=bar
    ```
2. ***KRM Function Runner Library***:

   A library to help execute KRM functions.

   Function execution library:
   - [X] Run multiple functions together
   - [X] Function can be either containerized image or an executable binary. 
   - [X] Function runtime integration is simplified.	

    ![krmlib (1)](https://user-images.githubusercontent.com/20452032/184198549-494cc2a6-f8e3-4560-b82b-48b047bd6476.png)
    
    * Input to the library is a custom Function object which defines the function configuration.
     ![image](https://user-images.githubusercontent.com/20452032/184207686-2b0bb7d2-c1e4-4fee-aae3-6d643d6f5b90.png)

    * Along with the function configuration it takes input resources against which these functions are executed. 
    * In response, the library emits a custom ResourceList which consists of transformed resource items and function results.
      ![image](https://user-images.githubusercontent.com/20452032/184207269-3afcfa32-4652-4de1-8763-a6e3870b61e6.png)

Presentation:
=============
[![image](https://user-images.githubusercontent.com/20452032/184214779-5f26e6aa-d4fa-4841-9a90-dfe94119447b.png)](https://www.youtube.com/watch?v=ynfLS5BE3a0)

References:
==========
- [Enhancement crane sub-command](https://github.com/MundraAnkur/enhancements/tree/master/enhancements/crane-2.0/run-functions)
- [Crane sub-command to run krm functions](https://github.com/konveyor/crane/pull/136)
- [Library to run KRM Fn images and binaries](https://github.com/konveyor/krm-lib/pull/1)
