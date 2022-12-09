---
title: "Making a plugin isn't so hard..."
excerpt: "In this post, I describe my experience with plugin creation for JetBrains Rider."
header:
  og_image: /assets/images/2021-03-25-plugin-for-rider/cover.jpg
categories: posts
author: Rival Abdrakhmanov
date: 2021-03-25
tags: ["Distributed application", "Project Tye", "Plugin", "JetBrains Rider", "ASP.NET Core"]
---

![Title image](/assets/images/2021-03-25-plugin-for-rider/cover.jpg)

It was always interesting for me is it difficult to build your plugin for IDE. Recently, I've decided to try. It turns out it's not that complicated, and I would like to share my experience.

_Disclaimer: I'm not a Kotlin or Java developer. Moreover, this is my first experience with these languages. So, the code below might be not fully optimized or doesn't follow best practices._

In this post, I want to show that plugin creation isn't rocket science and maybe inspire you to build your own. The more plugins we have, the more productive the whole community will be. This post isn't a how-to because JetBrains has detailed documentation, and you can find answers there. My goal is to demonstrate that there is a short step from an idea to a worth solution.

# Template

To start with, you could create a repository from an IntelliJ Platform Plugin Template. It simplifies the first setup, creates GitHub Actions for building and publishing plugin and more.

[IntelliJ Platform Plugin Template](https://github.com/JetBrains/intellij-platform-plugin-template)

After that, you'll have a new repository for your plugin. Let's clone it and add some logic.

# Project Tye

Lately, I published a post about [Project Tye](https://github.com/dotnet/tye). It's an experimental tool from Microsoft to develop distributed applications locally and deploy them to the Kubernetes cluster. So, I want to add support for this tool to the Rider.

[Distributed application with Project Tye]({% post_url 2020-10-18-distributed-application-with-project-tye %})

I'll briefly remind you how this tool works. First of all, let's create a test solution for our experiments.

```
$ dotnet new sln --name TyeExperiments
$ dotnet new web --name TyeApi
$ dotnet new worker --name TyeWorker
$ dotnet sln TyeExperiments.sln add TyeApi/TyeApi.csproj
$ dotnet sln TyeExperiments.sln add TyeWorker/TyeWorker.csproj
```

Now, we have a solution with two projects. Next, [install tye](https://github.com/dotnet/tye/blob/main/docs/getting_started.md) .NET tool. Note that you need first install [.NET Core 3.1](https://dotnet.microsoft.com/download/dotnet/3.1). After that, run the following command:

```
$ dotnet tool install -g Microsoft.Tye --version "0.6.0-alpha.21070.5"
```

Finally, go to the `TyeExperiments` solution folder and execute a command:

```
$ tye run
```

Tye starts your projects, and you can find them on the dashboard `http://127.0.0.1:8000/`.

![Tye dashboard](/assets/images/2021-03-25-plugin-for-rider/tye-dashboard.png)

# Run Configuration

Excellent, now what I want to accomplish is to run this command directly from my IDE. IntelliJ Platform SDK gives us the ability to add custom [Run Configuration](https://plugins.jetbrains.com/docs/intellij/basic-run-configurations.html). So, I think that it's the best place to extend basic functionality with our plugin. There is a [tutorial](https://plugins.jetbrains.com/docs/intellij/run-configurations.html) in the IntelliJ Platform documentation, and I show my implementation bellow.

Start a new extension from the `plugin.xml` file and register `configurationType`.

```xml
<extensions defaultExtensionNs="com.intellij">
  <configurationType implementation="com.github.rafaelldi.tyeplugin.run.TyeConfigurationType"/>
</extensions> 
```

Implement this [`ConfigurationType`](https://plugins.jetbrains.com/docs/intellij/run-configuration-management.html#configuration-type) with a new class.

```kotlin
class TyeConfigurationType : ConfigurationType {
    override fun getDisplayName() = "Tye"

    override fun getConfigurationTypeDescription() = "Tye run command"

    override fun getIcon() = AllIcons.General.Information

    override fun getId() = "TYE_RUN_CONFIGURATION"

    override fun getConfigurationFactories(): Array<ConfigurationFactory> = arrayOf(TyeConfigurationFactory(this))
}
```

Introduce a new [`ConfigurationFactory`](https://plugins.jetbrains.com/docs/intellij/run-configuration-management.html#configuration-factory) to be able to produce a run configuration.

```kotlin
class TyeConfigurationFactory(type: TyeConfigurationType) : ConfigurationFactory(type) {
    companion object {
        private const val FACTORY_NAME = "Tye configuration factory"
    }

    override fun createTemplateConfiguration(project: Project): RunConfiguration {
        return TyeRunConfiguration(project, this, "Tye")
    }

    override fun getName() = FACTORY_NAME

    override fun getId() = FACTORY_NAME
}
```

Create a [`RunConfiguration`](https://plugins.jetbrains.com/docs/intellij/run-configuration-management.html#run-configuration) class. With it, you can execute different actions, for example, call tye commands.

```kotlin
class TyeRunConfiguration(project: Project, factory: TyeConfigurationFactory, name: String) :
    RunConfigurationBase<TyeCommandLineState>(project, factory, name) {

    override fun getConfigurationEditor(): SettingsEditor<out RunConfiguration> = TyeSettingsEditor()

    override fun checkConfiguration() {
    }

    override fun getState(executor: Executor, environment: ExecutionEnvironment): RunProfileState {
        return TyeCommandLineState(environment, this, project)
    }
}
```

Additionally, I create a `TyeCommandLineState` class to locate operations with a command line. We'll come back to this later.

```kotlin
open class TyeCommandLineState(
    environment: ExecutionEnvironment,
    private val runConfig: TyeRunConfiguration,
    private val project: Project
) : CommandLineState(environment) {

    override fun startProcess(): ProcessHandler {
        val commandLine = GeneralCommandLine()
        val handler = OSProcessHandler(commandLine)
        return handler
    }
}
```

After all, add [`SettingsEditor`](https://plugins.jetbrains.com/docs/intellij/run-configuration-management.html#settings-editor) class to handle UI form.

```kotlin
class TyeSettingsEditor : SettingsEditor<TyeRunConfiguration>() {
    private lateinit var panel: JPanel

    override fun createEditor(): JComponent {
        createUIComponents()
        return panel
    }

    override fun resetEditorFrom(runConfig: TyeRunConfiguration) {
    }

    override fun applyEditorTo(runConfig: TyeRunConfiguration) {
    }

    private fun createUIComponents() {
        panel = JPanel().apply {
            layout = VerticalFlowLayout(VerticalFlowLayout.TOP)
        }
    }
}
```

Pretty straightforward implementation. I won't go into detail about each file. As I said before, you can find their description in the documentation. Eventually, if you run the plugin, you would see the new `Run Configuration` type. For now, it doesn't do anything; let's go to the next section and add some behaviour.

![Tye run configuration](/assets/images/2021-03-25-plugin-for-rider/tye-run-config.png)

# Tye run command

To call `tye run` command, we'll modify `TyeCommandLineState` class.

```kotlin
override fun startProcess(): ProcessHandler {
    val arguments = mutableListOf<String>()
    arguments.add("run")

    val commandLine = GeneralCommandLine()
        .withParentEnvironmentType(GeneralCommandLine.ParentEnvironmentType.CONSOLE)
        .withWorkDirectory(project.basePath)
        .withExePath("tye")
        .withParameters(arguments)

    val handler = OSProcessHandler(commandLine)
    handler.startNotify()
    ProcessTerminatedListener.attach(handler, environment.project)
    return handler
}
```

Start the plugin again, select `TyeExperiments` solution folder and run the new tye configuration type. You'll see the same logs as before when we ran tye independently.

![Tye logs](/assets/images/2021-03-25-plugin-for-rider/tye-logs.png)

# Conclusion

This post shows how to create a primitive plugin to envelope some command-line tool calls. Of course, the capabilities of the platform are much more significant. You can find a lot of complicated plugins in the [marketplace](https://plugins.jetbrains.com/). I've tried to show that if you want to add some simple functionality, it's not hard to do.

The documentation is an excellent starting point for your exploration. If it isn't enough, you can find how to implement some functionality in the [other plugins](https://plugins.jetbrains.com/intellij-platform-explorer/).

I hope I was able to motivate you to try creating your own plugin. My source code you can find on GitHub.

[Tye Plugin](https://github.com/rafaelldi/tye-plugin)

# References

* [Project Tye on GitHub](https://github.com/dotnet/tye)
* [Introducing Project Tye](https://devblogs.microsoft.com/aspnet/introducing-project-tye/)
* [IntelliJ Platform Plugin Template](https://github.com/JetBrains/intellij-platform-plugin-template)
* [IntelliJ Platform SDK](https://plugins.jetbrains.com/docs/intellij/welcome.html)
* [IntelliJ Platform Explorer](https://plugins.jetbrains.com/intellij-platform-explorer/)

*Image: Photo by Johanneke Kroesbergen-Kamps on Unsplash*
