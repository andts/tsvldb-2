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

In the existing configuration of the project, each generation task used the same folder as a target -- `generated/src`. 
This worked fine since each spec had a different package, so the actual code didn't mix. But apart from the code, 
the plugin, for some reason, also generated a pack of tooling-related files (like a maven pom.xml, some settings, etc). 
The code didn't mix due to different packages, but those files did, and each next task has overwritten the meta files 
from the previous.  

Because of that, for every build Gradle considered that the generated code (those meta-files specifically) 
did not match what was generated before, and it's right about time to go generate everything again.

## Example

So let's say we have two OpenAPI specs that we will use to generate client code

# The Crutch

A logical solution that comes to mind is to not generate those meta-files.   

To overcome this I had to do several things:
1. Try to disable generation of as much `meta` files as possible - we do not need all the project build files, we have them in the project already.
2. Generate code for each specification into a separate subfolder
3. Add the subfolder with generated code into the sourceSet to be picked up by Gradle
