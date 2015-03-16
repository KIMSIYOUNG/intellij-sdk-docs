---
title: External Builder API and Plugins
---

<!--
INITIAL_SOURCE https://confluence.jetbrains.com/display/IDEADEV/External+Builder+API+and+Plugins
-->

# {{ page.title }}

### External Build Process Workflow

When the user invokes an action that involves executing an external build (Make, Build Artifacts, etc.), the following steps happen:

*  Before-compile tasks are executed in the IntelliJ process.

*  Some source generation tasks that depend on the PSI (e.g. UI designer form to source compilation) are executed in the IntelliJ process.

*  BuildTargetScopeProvider extensions are called to calculate the scope of the external build (the set of build targets to compile based on the target module to make and the known set of changes).

*  The external build process is spawned (or an existing build process background process is reused).

*  The external build process loads the IntelliJ IDEA project model (.idea, .iml files and so on), represented by a ```JpsModel``` instance.

*  The full tree of targets to build is calculated, based on the dependencies of each build target to be compiled.

*  For each target, the set of builders capable of building this target is calculated.

*  For every target and every builder, the build() method is called. This can happen in parallel if the "Compile independent modules in parallel" option is enabled in the settings. For module-level builders, the order of invoking builders for a single target is determined by their category; for other builders, the order is undefined.

*  Caches to record the state of the compilation are saved.

*  Compilation messages reported through the CompileContext API are transmitted to the IntelliJ process and displayed in the UI (in the Messages view).

*  Post-compile tasks are executed in the IntelliJ process.

### Incremental Build

To support incremental build, the build process uses a number of caches which are persisted between build invocations. Even if your compiler doesn't support incremental build, you still need to report correct information so that incremental build works correctly for other compilers.

*  ```SourceToOutputMapping``` is a many-to-many relationship between source files and output files ("which source files were used to produce the specified output file"). It's filled by calls to BuildOutputConsumer.registerOutputFile() and ModuleLevelBuilder.OutputConsumer.registerOutputFile().

*  ```Timestamps``` records the timestamp of each source file when it was compiled by each build target. (If the compilation was invoked multiple times with different scopes and the file was changed inbetween, the last compiled timestamps for different build targets may vary.) It's updated automatically by JPS.

IntelliJ monitors the changes of the project content and uses the information from those caches to generate the set of dirty and deleted files for every compilation. (Dirty files need to be recompiled, and deleted files need to have their output deleted). A builder can also report additional files as dirty (e.g. if a method is deleted, the builder can report the classes using this method as dirty.) A module-level builder can add some files to the dirty scope; if this happens, and if the builder returns ```ADDITIONAL_PASS_REQUIRED``` from its build() method, another round of builder execution for the same module chunk will be started with the new dirty scope.

A builder may also want to have its custom caches to store additional information to support partial recompilation of a target (e.g. the dependencies between Java files in a module). To store this data, you can either store arbitrary files in the directory returned from ```BuildDataManager.getDataPaths().getTargetDataRoot()``` or use a higher-level API: ```BuildDataManager.getStorage()```

To pass custom data between the invocation of the same builder between multiple targets, you can use ```CompileContext.getUserData()``` and ```CompileContext.putUserData()```.

### Services and extensions in External Builder

The external builder process uses the standard Java
[services](http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html)
mechanism to support plugins. There are several service interfaces (e.g. BuilderService) which can be implemented in plugins to extend the builder functionality. An implementation of a service need to be registered by creating META-INF/services/<service-interface-fqn> file containing the qualified name of the implementation class. E.g. BuilderService's implementations are registered in META-INF/services/org.jetbrains.jps.incremental.BuilderService file. These files don't have extensions so you need to map corresponding patterns to text files in IDEA settings.

### Registering a plugin for External Builder

Sources of a plugin for External Builder should be put in a separate module. By convention such module has name '...-jps-plugin' and its sources are placed under 'jps-plugin' directory in the main plugin directory. Use compileServer.plugin extension to add the plugin to classpath of external build process, the plugin jar should be named "<jps module name>.jar". 'Build' | 'Prepare Plugin Module for deployment' action will automatically pack 'jps-plugin' part to a separate jar accordingly.

### Debugging a plugin for External Builder

Start IDEA with your plugin with the following VM option

```
-Dcompiler.process.debug.port=<port-number>
```


After that every time compilation is run in the test IDEA, the build  process will wait for debugger connection at this port and only then proceed.  In working copy of IDEA a "Remote" run configuration should be created and pointed it to this port. Specifying port "-1" will disable debugging mode.

### Profiling external build process

The build process has built-in self-cpu-profiling capabilities. To enable them do the following:

1. Copy <idea-home>/lib/yjp-controller-api-redist.jar and <idea-home>/bin/yjpagent.\*  files to <idea-system-dir>/compile-server

2. In "Settings \| Compiler \| Additional compiler process VM options" field enter \-Dprofiling.mode=true

3. Make sure automatic make is turned off

After this every build process run should result in a CPU snapshot stored in <user-home>/Snapshots directory.
Snapshots are named like "ExternalBuild\-\{date\}.snapshot".

Specifying \-Dprofiling.mode=false will turn profiling off.
Please capture a couple of snapshots for the situations in which you believe the build should work much faster than it works.

Please create an issue in the issue tracker and attach generated \*.snapshot files to it or upload them to
[ftp://ftp.intellij.net/.uploads](ftp://ftp.intellij.net/.uploads) and specify links in the issue.
Please also provide details about the memory and other VM settings for the build process you were using.


### Accessing External Build process' logs

The log file is located under the directory

```
<idea-system-directory>/log/build-log
```

There both "build-log.log" and "build-log.xml" files can be found.
The build-log.xml is a log4j configuration file, where the log level and desired logging categories can be adjusted.
This file contains logging from all  build sessions, including those from the auto-make.

### Accessing project model and configuration from External Build

The project model in External Build process is provided by JPS (JetBrains Project System).
A project is represented by JpsProject, a module by JpsModule and so on.
If your compiler depends on something that isn't added to the model yet (e.g. some facet settings),
you need to extend the JPS model (use JpsGwtModuleExtension as a reference implementation) and provide implementation of
JpsModelSerializerExtension to load the configuration from project files.

#### Implementing builder

If your compiler isn't involved into compilation of an existing BuildTarget you need to create a new implementation of BuildTarget and BuildTargetType. Also register an implementation of BuildTargetScopeProvider extension on IDEA side to add required targets to the build scope.
The builder implementation should extend either TargetBuilder or ModuleLevelBuilder class and should be created using BuilderService extension.

