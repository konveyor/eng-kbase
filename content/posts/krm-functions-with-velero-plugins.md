---
title: "Krm Functions With Velero Plugins"
date: 2022-08-04T09:03:34-06:00
draft: false
authors:
- Chaitanya Bhangale
tags:
- velero, KRM functions, openshift velero plugins, velero controller
---
### KRM Functions
The Kubernetes Resource Model (KRM) is a way to express the desired state that you want on your cluster. KRM functions small, interoperable, and language-independent executable programs packaged as containers that can be chained together as part of a configuration management pipeline. In a nutshell, they transform KRM resources in a predefined, holistic way.


### KRM Function Runner Library
The library takes in an input a image or an executable and a function config and runs KRM functions on a particular resource list. The input to library is in form of a ResourceList and the output we receive is a ResourceList. This Resouce list can be further parsed to add some additional conditions to a particular parameter and return the output from plugin accordingly.

## Solution
The solution of working this problem out was divided into two phases namely Feasibility Phase and Performance Improvement Phase.

### Feasibility phase
In this phase we explored if we can run KRM function from Openshift Velero Plugins. KRM functions were exported as executables and these executables were used to apply KRM functions from Openshift Velero Plugins.

For running KRM functions from velero plugins there are some changes to be made in DockerFile.ubi and the following lines are to be added. This copies the executables  to the pod while initializing the pod and we can use those executables for resource transformation. 

    VOLUME [ " /data" ]
    COPY /executables/ /data/executables

After these changes there are some additional changes that are to be made in openshift-velero-plugins code so as to bump the dependancies and use the krm runner lib to execute the KRM function binaries. The following code snippet shows an example of running a route-restore KRM function from the current openshift-plugins.

    func (p *RestorePlugin) Execute(input *velero.RestoreItemActionExecuteInput) (*velero.RestoreItemActionExecuteOutput, error) {
    
    	p.Log.Info("[route-restore] Entering Route restore plugin")
    	marshalledinput, _ := json.Marshal(input.Item)
    	//define functions that need to be passed to function runner
    	functions := []fn.Function{
    		{
    			Exec: "/data/executables/route",
    		},
    	}
    
    	//create a new function runner
    	//Working dir '/' cannot be used so we need to use Execution directory as /usr
    	runner := fn.NewRunner().
    			WithInput(marshalledinput).
    			WithFunctions(functions...).
    			WhereExecWorkingDir("/usr")
    
    	functionRunner, err := runner.Build()
    	if err != nil {
    		p.Log.Info("[route-restore] Error occured while building function runner")
    	}
    
    	//Excecute the binary
    	output, err := functionRunner.Execute()
    
    	//get the ouput resourcelist and parse host from it 
    	resource := output.Items[0].(*unstructured.Unstructured)
    	host,_,_ := unstructured.NestedString(resource.Object, "spec", "host")
    
    	if host == "" {
    		p.Log.Info("[route-restore] Stripping src cluster host from Route")
    		return velero.NewRestoreItemActionExecuteOutput(resource), nil
    	}
    
    	p.Log.Info("[route-restore] Route has statically-defined host so leaving as-is")
    	return velero.NewRestoreItemActionExecuteOutput(input.Item), nil
    }

The overall process can be demonstrated by the diagram below

![Run KRM functions from Velero Plugins](/krmfunctions-velero.png)


The left part of the diagram shows modification of Velero pod docker file and addition of executables in the velero container. We make use of this new Velero pod for backup/restore.The right part shows Velero openshift plugins that make use of KRM function runner library and also plugin KRM functions repository for transformation of resources to complete the backup restore process. DPA(Data Protection Application) needs to be changed to use images for new Velero pod and also add overriden changes for Openshift Velero plugins container as well. 

### Performance improvement phase
This Phase was development of a new plugin type BackupMultiItems and RestoreMultiItems to the Velero Plugins that will allow a plugin author to take in a list of resources, execute KRM functions, and return a list of items with translated data to be included in the backup. After adding a new plugin to accept a list of resources, modify the Velero Controller to pass in to this plugin type a list of resources so that we achieve performance enhancement by reducing the amount of plugin calls.

### Demo
The following demo shows an example of how to backup restore an application with the plugins modified to run KRM functions .The repositories and code which is modified is also highlighted in the demo video.

[Demo video]("https://www.youtube.com/watch?v=kuDWpvoHvbo" "Demo video")


### References
[Openshift-Velero-Plugin-Changes](https://github.com/openshift/openshift-velero-plugin/pull/160 "Openshift-Velero-Plugin-Changes")

[Velero Controller code](https://github.com/vmware-tanzu/velero "Velero Controller code")

[KRM Plugin Functions Repository](https://github.com/chaitanyab2311/plugin-krm-functions "KRM Plugin Functions Repository")







