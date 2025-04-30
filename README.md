# packages-example-npm

A "Hello world" NPM project with a GitHub Actions workflow to build and publish

## File Overview

Here is a brief explanation of the primary files and directories in this repository:

-   `.npmrc`: This is a configuration file for npm. It can be used to specify several config settings for npm, like the registry to download packages from when you run `npm install`. More information can be found in the [npmrc documentation](https://docs.npmjs.com/cli/configuring-npm/npmrc).
    
-   `index.js`: This is the main entry point for your application. It is the first script that gets executed when you run your application.
    
-   `package-lock.json`: This is an automatically generated file that is used to specify exact versions of the npm dependencies that your project uses. This helps to ensure that you're using the same versions of your dependencies across different environments. More information can be found in the [npm documentation](https://docs.npmjs.com/cli/configuring-npm/package-lock-json).
    
-   `package.json`: This file serves as the manifest for your application. It includes metadata about your project (like its name and version) and its dependencies. It also specifies scripts that can be run with `npm run`. More information can be found in the [npm documentation](https://docs.npmjs.com/cli/configuring-npm/package-json).

- `node_modules`: This is the directory where NPM installs your project's dependencies. The contents of this folder are automatically generated based on the `package.json` file and `package-lock.json` file. You typically don't interact with the contents of this directory directly and it's often ignored in source control (e.g. by adding it to the `.gitignore` file) because it can be recreated by running `npm install`. More information can be found in the [official NPM documentation](https://docs.npmjs.com/cli/configuring-npm/folders).
    
-   `.github/workflows/build-and-publish.yml`: This is a GitHub Actions workflow file. It defines a set of actions that should be automatically performed when certain events occur in this repository, like when code is pushed. More information can be found in the [GitHub Actions documentation](https://docs.github.com/en/actions).

## More helpful docs:

[Publishing Node.js packages](https://docs.github.com/en/actions/publishing-packages/publishing-nodejs-packages)

[Working with the npm registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-npm-registry)

[How to Create and Publish an NPM Package – a Step-by-Step Guide (Third-party guide)](https://www.freecodecamp.org/news/how-to-create-and-publish-your-first-npm-package/)
