# MovieHub -- Angular Micro Frontend Learning Notes

> A running log of the setup completed so far. This document records the
> decisions, commands, and reasons behind them so the project can be
> resumed later.
>
> **Current status:** The `moviehub-shell` Angular application runs on
> `http://localhost:4200`. Native Federation v4 has been installed and
> initialized with the Shell configured as a **dynamic host**. The
> remote applications have not been created yet.

## 1. Project goal and scope

We are building **MovieHub**, a movie-discovery application to learn
Micro Frontend Architecture with Angular.

### Initial features

-   Display currently playing / latest movies using The Movie Database
    (TMDB) API.
-   Show movie posters, titles, and ratings in a list or grid.
-   When a user selects a movie, open a modal dialog.
-   Show the selected movie's poster, rating, release year, cast, and
    overview/synopsis in the modal.
-   Add genre filtering later; it is not part of the initial
    implementation.

This is **not** a movie-booking application. We will not implement seat
availability or booking flows.

### Planned application architecture

We plan to create three separate Angular applications:

  --------------------------------------------------------------------------
  Application        Role                      Planned port Responsibility
  ------------------ ---------------- --------------------- ----------------
  `moviehub-shell`   Host / Shell                      4200 Application
                                                            frame,
                                                            navigation,
                                                            remote loading,
                                                            and modal
                                                            lifecycle

  `movie-listing`    Remote                            4201 Fetch and
                                                            display movies;
                                                            notify the Shell
                                                            when a movie is
                                                            selected

  `movie-details`    Remote                            4202 Display details
                                                            for the selected
                                                            movie
  --------------------------------------------------------------------------

The Shell will own the modal container and its open/close state. The
Movie Details remote will own the movie-specific content displayed
inside it. The Movie Listing remote will send the selected movie ID to
the Shell. The details remote can then retrieve the corresponding movie
details and credits.

We will pass the **movie ID**, rather than coupling the remotes by
passing an entire movie object.

## 2. Technology decisions

The current plan is:

-   Angular 21
-   Native Federation v4 (`@angular-architects/native-federation-v4`)
-   TypeScript
-   SCSS
-   Separate Angular application folders/workspaces
-   TMDB API for movie data

### Why separate Angular applications?

We chose separate applications rather than one multi-project Angular
workspace because this makes the boundaries between the Shell and each
Remote easier to see and practise:

-   Each application has its own `package.json` and `angular.json`.
-   Each application can have its own development server and build.
-   Each Remote can expose its own functionality.
-   The Shell consumes functionality from the Remotes through
    federation.

Separate folders alone do not automatically guarantee independent
deployment; deployment configuration will be covered later.

## 3. Environment checks

The machine initially reported:

``` text
Node.js:       v24.13.1
npm:           11.15.0
Global CLI:    Angular CLI 16.2.16
OS:            Windows 64-bit
```

We decided to use Angular 21 for this new project rather than Angular
17, while retaining the existing global Angular CLI 16 installation.

Node.js 24 was retained. We did not switch Node versions.

### Important version note

The Shell's dependency tree later reported:

``` text
@angular/cli:             21.2.31
@angular/build:           21.2.31
@angular/core:            21.2.26
@angular/compiler:        21.2.26
@angular/compiler-cli:    21.2.26
typescript:                5.9.3
```

The installed Angular framework packages are on the 21.2 patch line. The
CLI/build patch numbers are slightly higher. We verified the installed
dependency tree, but should continue checking package compatibility if a
later install or build reports a version constraint.

## 4. Outer project folder and Angular CLI installation

We created an outer folder named `MovieHub` and installed Angular CLI 21
locally in it:

``` bash
mkdir MovieHub
cd MovieHub

npm install --save-dev @angular/cli@21
```

We checked the local CLI with:

``` bash
npx ng version
```

### Why use `npx ng`?

-   `ng` normally invokes the globally installed Angular CLI.
-   `npx ng` can invoke the CLI installed locally in the current project
    directory.
-   This allowed us to use Angular CLI 21 without uninstalling or
    replacing the global Angular CLI 16.

The outer `MovieHub` folder currently has root-level `node_modules`,
`package.json`, and `package-lock.json` from this CLI installation. We
decided to **keep these files for now** rather than delete them during
setup.

## 5. Create the Shell application

From the outer `MovieHub` directory, we created the first Angular
application:

``` bash
npx ng new moviehub-shell
```

The application was created in its own folder:

``` text
MovieHub/
├── moviehub-shell/
│   ├── src/
│   ├── angular.json
│   ├── package.json
│   └── ...
├── node_modules/
├── package.json
└── package-lock.json
```

The Shell is the host application. It will provide the common
application frame and coordinate the loading of the remote applications.

We verified that the Angular application ran on:

``` text
http://localhost:4200
```

The application is running successfully.

## 6. TypeScript configuration issue and resolution

During setup, VS Code displayed a TypeScript diagnostic in
`moviehub-shell/tsconfig.app.json` relating to `rootDir`.

The application configuration now includes:

``` json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/app",
    "rootDir": "./src",
    "types": []
  },
  "include": [
    "src/**/*.ts"
  ],
  "exclude": [
    "src/**/*.spec.ts"
  ]
}
```

The important setting added in the file is:

``` json
"rootDir": "./src"
```

### What does `rootDir` do?

It tells TypeScript the root directory of the source files for this
compilation and helps determine the emitted output directory structure.

The base `tsconfig.json` is extended by `tsconfig.app.json`. The base
configuration includes strict TypeScript and Angular template checks.

The diagnostic is now gone, and the app runs. Do not add or change this
setting again unless a new compiler diagnostic gives a reason to do so.

### TypeScript version check

We ran:

``` bash
npm ls typescript
```

The result showed TypeScript `5.9.3`, deduplicated in the dependency
tree.

We also checked the Angular package versions with:

``` bash
npm ls @angular/core @angular/compiler @angular/compiler-cli @angular/cli typescript
```

The versions are recorded in the Environment section above.

## 7. Install and initialize Native Federation in the Shell

We chose Native Federation v4 for Angular 21.

From inside the Shell application directory:

``` bash
cd moviehub-shell
```

We installed the Native Federation package:

``` bash
npm install --save-dev @angular-architects/native-federation-v4@21
```

Then we initialized the Shell as a **dynamic host**:

``` bash
npx ng generate @angular-architects/native-federation-v4:init --project moviehub-shell --port 4200 --type dynamic-host
```

### What does `dynamic-host` mean?

The Shell is configured to consume remote applications. It uses a
runtime manifest to look up the URLs of those remote applications.

A host does not need an `exposes` entry merely because it is a host. The
`exposes` configuration is normally needed in a Remote to identify which
modules it makes available to other applications. An application can be
both a host and a remote if its architecture requires both roles.

## 8. Generated federation files inspected so far

### `public/federation.manifest.json`

The manifest is in the Shell's `public` folder. It is currently empty:

``` json
{}
```

This is expected at this stage because the remote applications have not
yet been created and registered.

Later, it will contain entries similar to the following (the URLs are
planned local development examples):

``` json
{
  "movie-listing": "http://localhost:4201/remoteEntry.json",
  "movie-details": "http://localhost:4202/remoteEntry.json"
}
```

The manifest lets the dynamic host discover remote entry URLs at
runtime. We will confirm the exact generated remote entry filename and
configuration when we initialize the Remote applications.

### `federation.config.mjs`

The generated Shell configuration currently includes:

``` javascript
import {
  withNativeFederation,
  fromPackageJson,
} from '@angular-architects/native-federation-v4/config';

export default withNativeFederation({
  name: 'moviehub-shell',

  //shareScope: autoShareScope({level: 'minor'}),

  shared: fromPackageJson({
    singleton: true,
    strictVersion: true,
    requiredVersion: 'auto',
    build: 'package',
  }).patch(['@angular/*'], { includeSecondaries: { keepAll: true } }),

  //sharedMappings: mappingsFromWorkspace({ singleton: true }).filter(['@my-org/*']).get(),

  skip: ['rxjs/ajax', 'rxjs/fetch', 'rxjs/testing', 'rxjs/webSocket'],

  features: {
    denseChunking: true,
    denseExternals: true,
    strictImports: false,
  },
});
```

Important observations:

-   `name` identifies this federation application as `moviehub-shell`.
-   `shared` configures dependency sharing based on package metadata.
-   `singleton: true` requests a single shared instance for the
    dependency in the federation share scope.
-   `strictVersion: true` enforces the configured version compatibility
    rules.
-   `requiredVersion: 'auto'` derives the required version from package
    metadata.
-   `skip` lists RxJS entry points that should not be shared through
    this configuration.
-   `features` contains Native Federation v4 options. We will explain
    and revisit these when we understand the generated build/runtime
    setup.

Do not edit these generated options casually. We will make changes when
a specific integration requirement calls for them.

### Why is the file `.mjs`, not `.js`?

`.mjs` explicitly identifies the file as an ECMAScript Module. That
format supports the `import` and `export` syntax used in the
configuration.

A `.js` file can also be interpreted as an ES module when its package
context declares `"type": "module"`, but `.mjs` makes the module format
explicit.

### Why is there no `exposes` entry in the Shell?

The Shell was initialized as a dynamic host, whose initial job is to
consume remote modules. The Movie Listing and Movie Details applications
will be initialized as remotes, and their federation configurations will
declare the modules they expose.

## 9. Current project status

Completed:

-   [x] Defined MovieHub's movie-discovery scope.
-   [x] Chose Angular 21 and Native Federation v4.
-   [x] Created the outer `MovieHub` folder.
-   [x] Installed Angular CLI 21 locally in the outer folder.
-   [x] Created the `moviehub-shell` Angular application.
-   [x] Resolved the `tsconfig.app.json` `rootDir` diagnostic.
-   [x] Verified TypeScript and Angular dependency versions.
-   [x] Ran the Shell successfully on port 4200.
-   [x] Installed Native Federation v4 in the Shell.
-   [x] Initialized the Shell as a dynamic host.
-   [x] Inspected `federation.config.mjs` and
    `public/federation.manifest.json`.

Not completed yet:

-   [ ] Create `movie-listing`.
-   [ ] Initialize `movie-listing` as a Native Federation remote.
-   [ ] Configure the Movie Listing exposed module.
-   [ ] Create `movie-details`.
-   [ ] Initialize `movie-details` as a Native Federation remote.
-   [ ] Configure the Movie Details exposed module.
-   [ ] Register both remotes in the Shell's federation manifest.
-   [ ] Load the Movie Listing remote in the Shell.
-   [ ] Integrate TMDB API calls.
-   [ ] Pass the selected movie ID from Listing through the Shell to
    Details.
-   [ ] Display the Movie Details remote in a modal.
-   [ ] Add later enhancements such as genre filtering.
-   [ ] Build and test the applications independently.

## 10. Next implementation step

Create the Movie Listing application as a separate Angular application,
from the outer `MovieHub` directory:

``` bash
npx ng new movie-listing --routing --style=scss
```

Choose the appropriate Angular CLI prompts for the project. Then verify
its Angular version and confirm that it builds and runs before adding
Native Federation.

After that, initialize it as a **remote** using the Native Federation v4
schematic. We will inspect the generated remote configuration and its
`exposes` entry before connecting it to the Shell.

------------------------------------------------------------------------

## Key concepts to remember

  -----------------------------------------------------------------------
  Term                                Meaning in MovieHub
  ----------------------------------- -----------------------------------
  Shell / Host                        Main application that loads remote
                                      modules

  Remote                              Independently built application
                                      that exposes modules

  Exposes                             Public module entry points provided
                                      by a Remote

  Manifest                            Runtime mapping from remote names
                                      to remote entry URLs

  Shared dependencies                 Packages configured for reuse
                                      across federated applications

  Dynamic host                        Host that discovers remote
                                      applications using runtime
                                      configuration

  Native Federation                   Mechanism used to integrate
                                      independently built applications
                                      through native browser module
                                      loading
  -----------------------------------------------------------------------

**Learning principle:** For each configuration we add, understand three
things: what it configures, why the architecture needs it, and which
application owns that responsibility.
