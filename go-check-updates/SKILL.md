---
name: go-check-updates
description: Update go dependencies. Use when user ask "update go dependencies" or "update go modules".
---

Your task is to update go dependencies in the project. Before making any updates, you should first analyze the project structure to understand where the `go.mod` file(s) is located all dependecies:
1. if the project is a multimodule project or not;
2. if the project uses internal submodules as dependencies if other modules;

The use one of the following strategies to update the dependencies. Before running any startegy explicitly tell to user the plan for the update. Always make changes in a new branch and make pull request using GitHub MCP. 

## Strategy 1: Single go.mod file
If the project has a single `go.mod` file, you can simply run the command `go get -u` in the root directory of the project to update all dependencies to their latest versions. After updating, make sure to run `go mod tidy` to clean up any unused dependencies.

If the project contains `version.go` file with version information, you should also update the version information in that file to reflect the new versions by incrementing the patch version number (e.g., from `v1.0.0` to `v1.0.1`).

## Strategy 2: Multiple go.mod files (multimodule project) no usage of internal submodules as dependencies
If the project has multiple `go.mod` files, you will need to check each module's `go.mod` file for dependencies and update them accordingly. You can navigate to each module's directory and run `go get -u` to update the dependencies for that module. After updating, run `go mod tidy` in each module's directory to clean up any unused dependencies.

If the submodules project contains `version.go` file with version information, you should also update the version information in that file to reflect the new versions by incrementing the patch version number (e.g., from `v1.0.0` to `v1.0.1`).


## Strategy 3: Multiple go.mod files (multimodule project) with usage of internal submodules as dependencies
If the project has multiple `go.mod` files and uses internal submodules as dependencies, you will need to build a dependency graph to understand the relationships between the modules and their dependencies. This will help you determine the order in which to update the modules and their dependencies. You can use tools like `go mod graph` to visualize the dependency graph. Once you have understood the dependency graph, tell this to user and then proceed.

The update process is complicated it requires careful coordination for each layer. Start with the leaf modules (the ones that do not have any other modules depending on them) and update their dependencies first. After updating the leaf modules, move up the dependency graph to update the modules that depend on the leaf modules, and so on until you reach the root module.   

On each layer the update process is following:
1. Use `go get -u` to update the dependencies for the modules in that layer. 
2. Explicitly update internal module references using `go get module@version` (not just `go get -u`).
3. After updating, run `go mod tidy` in each module's directory to clean up any unused dependencies.
4. If the submodules project contains `version.go` file with version information, you should also update the version information in that file to reflect the new versions by incrementing the patch version number (e.g., from `v1.0.0` to `v1.0.1`).
5. Make pull request using GitHub MCP for the changes in that layer.
6. Either merge it automatically when all checks are passed or ask user to merge it.
7. CI/CD makes automatic tagging and release of the updated module. If the project doesn't have automated tagging (you can check if `.github/workflows/build.yml` exists and automatic tagging is enabled), manually create and push tags: `git tag v0.11.2 && git push github v0.11.2`. 
8. Wait for tags to be available before proceeding to next layer.

Try to batch multiple depdendecies updates of same layer in a single pull request if possible to minimize the number of pull requests and merges.

