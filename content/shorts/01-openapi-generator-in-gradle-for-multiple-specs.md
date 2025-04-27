+++
title = "How to generate sources from multiple OpenAPI specifications with Gradle"
date = "2025-04-06"
updated = "2025-04-06"
+++

This is a short note on how to setup a Java Gradle project that uses OpenApi Generator plugin 
to generate client/server code for multiple specifications in such a way that the code will not
be regenerated on every project build.  

{{ admonition(type="info", title="Is this fixed already?", text="This is an issue I faced on a project 
in late 2024, so no guarantees if this is still relevant.") }}

# Problem

On a project that uses Gradle as a build system, there was an issue when all client/server code
generated from OpenApi specs was regenerated for every build. It is expected when doing a full rebuild
(a `clean build` locally or in a CI pipeline), but why would you want that when you are just running
your unit tests? This made the process of writing unit tests unbearable.

*I still don't know if the projects' lack of tests was the result of this,
or nobody bothered to fix this issue due to the lack of tests* 🤔

# The Cause

In the existing configuration of the project, each generation task used the same folder as a target -- `build/generated/openapi`. 
This worked fine since each spec had a different package, so the actual code didn't mix. But apart from the code, 
the plugin, for some reason, also generated a pack of tooling-related files (like a maven pom.xml, etc). 
The code didn't mix due to different packages, but those files did, and each next task has overwritten the meta files 
from the previous.  

Because of that, for every build Gradle considered that the generated code (those meta-files specifically) 
did not match what was generated before, and it's right about time to go generate everything again.

## Example

Let's reproduce the issue first. We have two OpenAPI specs that we will use to generate server API code.

Ths full source of example is available here - . The code is the same, but Jas several more classes so it would compile.

With this configuration, each time we run `Gradle build` the OpenApi generator tasks will execute, even though the specs didn't change. Let's try to fix that.

# The Crutch

A logical solution that comes to mind is to disable generation of those project files. Unfortunately, all I could find are issues that request this feature.

Instead, we will do two things - exclude as much project files as possible from generation, and extract each API into its own folder. Technically, you could only do the second, and it would solve the issue, but I wanted to keep the sources as clean as possible.

## Cleanup

The only working way to remove unnecessary project files happens to be through `.openapi-ignore-files`. 
Put a file like this into the root (or wherever you like):

This file excludes from generation all files that are not in the source directory. Exactly what we needed.

But there is one issue still which requires the second step. There are two metadata file in .OpenApi folder files and version, that contain list of all generated files and version of the generator accordingly. And when several tasks generate sources - they will overwrite those files. Let's continue.

## Extraction

Now let's put every API interface into its own folder.
We will change the task configuration a bit, and will write some more config to attach the sources to the sourceset.

The approach here is automated, so you don't have to add the sources of every task but hand.

Note we had to disable the default task, so it wouldn't mix with our actual generator tasks.

# Summary

You can try this approach if you encounter similar issues in your projects 
To reiterate, you might need this if:
- you build using Gradle
- you have several OpenApi specs
- you build often