# Deploy Angular Universal App with Firebase
This step-by-step tutorial will show you how to deploy a Angular App with server-side rendering using Angular Universal with Firebase Hosting.  This project was generated with [Angular CLI](https://github.com/angular/angular-cli) version 1.6.2.
* Angular version: 5.0.0
* Firebase CLI version: 3.16.0

# Generate a Universal Angular App
Angular CLI has native Universal support starting from v1.6.  We will use the CLI to quickly generate Angular Universal server files, and then make some minor changes for our production build.

1. Create a new Angular project
    ```
    ng new angular-universal-firebase
    ```

2. Generate Angular Universal using Angular CLI (v1.6 or greater)
    ```
    ng generate universal universal
    ```

3. Install `@angular/platform-server`
    ```
    yarn add @angular/platform-server
    ```

4. Modify `main.server.ts` to the following:
    ```typescript
    import { enableProdMode } from '@angular/core';
    export { AppServerModule } from './app/app.server.module';

    enableProdMode();
    ```

5. Add `/dist-server` to `.gitignore`
    ```git
    # compiled output
    /dist
    /dist-server
    ...
    ``` 

6. Build the app (`/dist` folder) and the server to render the app (`/dist-server` folder).
    ```
    ng build --prod && ng build --prod --app universal --output-hashing=none
    ```

# Deploying to Firebase
Since we now have an Angular app with a `/dist` and `/dist-server` directories, we will now use Firebase Cloud Functions to serve our application.  This guide was originally written by *Aaron Te* and can be found at [Hackernoon: Deploy Angular Universal w/ Firebase](https://hackernoon.com/deploy-angular-universal-w-firebase-ad70ea2413a1), but has been slightly modified with minor changes.

1. Create a Firebase project (eg. `angular-universal-firebase`)

2. Log in to firebase using 
    ```
    firebase login
    ```

3. Initialize Firebase in the Angular project
    ```
    firebase init
    ```
    * Select `Functions` and `Hosting` for features
    * Select the firebase project you created (eg. `angular-universal-firebase`)
    * Select `javascipt` as the language used to write Cloud Functions
    * Select `no` to install dependencies with npm
    * Select all defaults for `Hosting`

4. Add Angular dependencies to `functions/package.json`, including @angular, rxjs, and zone.js.  The easiest way to add these dependencies will be to copy them from your root `package.json` file.  **IMPORTANT: Install dependencies in the `functions` directory with yarn.  NPM does not properly install `firebase-admin`**.  You will have to install express using `yarn add express`.
    ```json
    "dependencies": {
        "@angular/animations": "^5.0.0",
        "@angular/common": "^5.0.0",
        "@angular/compiler": "^5.0.0",
        "@angular/core": "^5.0.0",
        "@angular/forms": "^5.0.0",
        "@angular/http": "^5.0.0",
        "@angular/platform-browser": "^5.0.0",
        "@angular/platform-browser-dynamic": "^5.0.0",
        "@angular/platform-server": "^5.1.2",
        "@angular/router": "^5.0.0",
        "express": "^4.16.2",
        "firebase-admin": "~5.4.2",
        "firebase-functions": "^0.7.1",
        "rxjs": "^5.5.2",
        "zone.js": "^0.8.14"
    },
    ```

5. Install all dependencies in the `functions` directory using `yarn`.
    ```
    yarn install
    ```

6. Copy the `dist` and `dist-server` folders into the `functions` directory. This is because Firebase functions cannot access files outside of this directory.  There should now be exact copies of those two folders in `functions/dist` and `functions/dist-server`, respectively.

7. Create Firebase function (`index.html`) to serve the app.  This file is found in the `functions` directory.
    ```javascript
    require('zone.js/dist/zone-node');

    const functions = require('firebase-functions');
    const express = require('express');
    const path = require('path');

    const { enableProdMode } = require('@angular/core');
    const { renderModuleFactory } = require('@angular/platform-server');
    const { AppServerModuleNgFactory } = require('./dist-server/main.bundle');

    enableProdMode();

    const index = require('fs')
        .readFileSync(path.resolve(__dirname, './dist/index.html'), 'utf8')
        .toString();

    let app = express();

    app.get('**', function (req, res) {
        renderModuleFactory(AppServerModuleNgFactory, {
            url: req.path,
            document: index
        }).then(html => res.status(200).send(html));
    });

    exports.ssr = functions.https.onRequest(app);
    ```

8. Update `firebase.json` to:
    ```json
    {
        "hosting": {
            "public": "dist",
            "ignore": [
                "firebase.json",
                "**/.*",
                "**/node_modules/**"
            ],
            "rewrites": [{
                "source": "**",
                "function": "ssr"
            }]
        }
    }
    ```

9. Delete the `/public` folder that was automatically generated by Firebase functions during the `firebase init` process
    ```
    rm -rf public
    ```

10. Delete `dist/index.html` from the root directory.  This is so Firebase won’t serve the html file but rather run the ssr function. 

11. Add `functions/dist`, `functions/dist-server` and `functions/node_modules` to `.gitignore`
    ```git
    # compiled output
    /dist
    /dist-server
    /functions/dist
    /functions/dist-server
    ...

    # dependencies
    /node_modules
    /functions/node_modules
    ```

12. Deploy to Firebase
    ```
    firebase deploy
    ```

# Further help
To get more help on the Angular CLI use `ng help` or go check out the [Angular CLI README](https://github.com/angular/angular-cli/blob/master/README.md).
