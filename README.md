# Creating standalone Desktop Applications with React, Electron.


There are quite a few tutorials on the Internet that cover the process of setting up React inside an Electron app but very few (if any) cover the solutions to common problems you run in to when packaging the app for distribution. I am going to run you through how I setup a production ready work flow for creating a desktop app with React in Electron 


## Let’s get started.

```cmd
yarn create react-app MyDesktopApp
```

```cmd
yarn add electron electron-builder nodemon concurrently wait-on -D
```

 Now that we have React ready to go, let’s setup Electron. We’ll need to install a few packages for that.

 <b>Development Dependencies:</b>

- electron — well .. Electron.
- electron-builder — To package the Electron app.
- nodemon — To monitor for changes during development and hot reload.
- concurrently — To launch the React app and Electron together
- wait-on — To wait for the React app to launch before the Electron app is launched.

 <b>Dependencies:</b>

- electron-is-dev — Simple library to check if we’re in development mode or not.
- cross-env.

```cmd
yarn add electron electron-builder nodemon concurrently wait-on -D
```
```cmd
yarn add electron-is-dev cross-env 
```

In your public folder, create a two new files called electron.js and preload.js.


# electron.js file will have the following code.

```javascript
const { app, BrowserWindow, ipcMain } = require('electron'); // electron
const isDev = require('electron-is-dev'); // To check if electron is in development mode
const path = require('path');
let mainWindow;
// Initializing the Electron Window
const createWindow = () => {
  mainWindow = new BrowserWindow({
    width: 600, // width of window
    height: 600, // height of window
    webPreferences: {
      // The preload file where we will perform our app communication
      preload: isDev 
        ? path.join(app.getAppPath(), './public/preload.js') // Loading it from the public folder for dev
        : path.join(app.getAppPath(), './build/preload.js'), // Loading it from the build folder for production
      worldSafeExecuteJavaScript: true, // If you're using Electron 12+, this should be enabled by default and does not need to be added here.
      contextIsolation: true, // Isolating context so our app is not exposed to random javascript executions making it safer.
    },
  });
	// Loading a webpage inside the electron window we just created
  mainWindow.loadURL(
    isDev
      ? 'http://localhost:3000' // Loading localhost if dev mode
      : `file://${path.join(__dirname, '../build/index.html')}` // Loading build file if in production
  );
	// Setting Window Icon - Asset file needs to be in the public/images folder.
  mainWindow.setIcon(path.join(__dirname, 'images/appicon.ico'));
	// In development mode, if the window has loaded, then load the dev tools.
  if (isDev) {
    mainWindow.webContents.on('did-frame-finish-load', () => {
      mainWindow.webContents.openDevTools({ mode: 'detach' });
    });
  }
};
// ((OPTIONAL)) Setting the location for the userdata folder created by an Electron app. It default to the AppData folder if you don't set it.
app.setPath(
  'userData',
  isDev
    ? path.join(app.getAppPath(), 'userdata/') // In development it creates the userdata folder where package.json is
    : path.join(process.resourcesPath, 'userdata/') // In production it creates userdata folder in the resources folder
);
// When the app is ready to load
app.whenReady().then(async () => {
   createWindow(); // Create the mainWindow
});
// Exiting the app
app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});
// Activating the app
app.on('activate', () => {
  if (mainWindow.getAllWindows().length === 0) {
    createWindow();
  }
});
// Logging any exceptions
process.on('uncaughtException', (error) => {
  console.log(`Exception: ${error}`);
  if (process.platform !== 'darwin') {
    app.quit();
  }
});
```

Go to package.json and add the following.


```javascript 
  "main": "./public/electron.js",
  "homepage": "./",
  ```

  Now we need a few scripts in package.json to be added. 

```javascript 
    "electron:serve": "concurrently -k \"cross-env BROWSER=none yarn start\" \"yarn electron:start\"",
    "electron:build": "yarn build && electron-builder -c.extraMetadata.main=build/main.js",
    "electron:start": "wait-on tcp:3000 && electron ."
```
   Your scripts should look something like this.

```javascript
 "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject",
    "electron:serve": "concurrently -k \"cross-env BROWSER=none yarn start\" \"yarn electron:start\"",
    "electron:build": "yarn build && electron-builder -c.extraMetadata.main=build/main.js",
    "electron:start": "wait-on tcp:3000 && electron ."
  },
```
<details><summary>To explain what they do </summary>
<p>

- start-react — Will start just the React app only
- build-react — Will build the React app only
- start-electron — This will use nodemon to watch for changes in the public folder and then execute electron. If you want to add more folders to be monitored, just add - another —-watch followed by the path to that folder.
- dev — Will first run React, wait for it to boot up and then start Electron.
- postinstall — It will make electron-builder install any dependencies we need for our app
- pack-app — Building the app can take time. Packing the app is shorter. It’ll just pack the app so you can test your production builds.
- build- Builds your app for distribution
- test — Comes with Create React App. Testing for your React app.
- eject — Comes with Create React App. Ejects your app from the CRA pipeline.

</p>
</details>


# Communication Between React & Electron
So in order for these two sides to be able to talk to each other, we create a bridge. That might sound complicated but it really is not.

What you are doing is basically creating a simple API inside your preload.js by defining simple functions that are preloaded to the app. So the front end only has access to these functions only making your app secure.

preload.js
```javascript 
const { ipcRenderer, contextBridge } = require('electron');
contextBridge.exposeInMainWorld('api', {
  // Invoke Methods
  testInvoke: (args) => ipcRenderer.invoke('test-invoke', args),
  // Send Methods
  testSend: (args) => ipcRenderer.send('test-send', args),
  // Receive Methods
  testReceive: (callback) => ipcRenderer.on('test-receive', (event, data) => { callback(data) }
});
```
sending messages to the backend:
```javascript 
window.api.testSend("YourMessage")
```
in the backend:
```javascript
  //IPC MAIN
  ipcMain.on("test-send", (event, args) => {
    console.log(args)
  })
  
 ```
 
 
 Start the App:
 
```cmd 
 npm run electron:serve
 ```
 
 # original source <a href="https://wykrhm.medium.com/creating-standalone-desktop-applications-with-react-electron-and-sqlite3-269dbb310aee">Here</a>

 
 
 
