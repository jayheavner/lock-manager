# lock-manager
z-wave lock manager for home assistant built on node-red using docker

# Installation
I'm using Docker and Docker Compose. It's not required. You could simply import the flow, add the palette packages, and make the changes to the settings.

1. Add auth to settings.js - Find the adminAuth section in settings.js. Add the username and generate a hashed password. See the [node-red documentation](https://nodered.org/docs/user-guide/runtime/securing-node-red) for more info.
```
adminAuth: {
  type: 'credentials',
  users: [
    {
      username: 'my user',
      password: '<password hash>',
      permissions: '*'
    }
  ]
},
```

2. Add the following packages to palette. Hamburger menu in the top-right corner, 'Manage palette', click the 'Install' tab. Find the package and install. *Restart node-red once everything has been installed*
    -	node-red-contrib-mongodb3
    -	node-red-dashboard
    -	node-red-node-ui_list
    -	node-red-contrib-home-assistant-websocket
    -	node-red-contrib-credentials

3. Import the flow.
4. Update Home Assistant Configuration node. *Note: The node must be authorized. Currently, the preferred way is to generate/use a long living access token in HA under user profile and enter it in the settings.*
5. Install missing npm packages. This project is using crypto-js, moment, and lodash. To install, get to a bash prompt and navigate to the working folder and type `npm install --save`
6. Add these packages to setting.js. Find the functionGlobalContext section in settings.js and add the following.
  ```
    cryptojs: require('crypto-js'),
    lodash: require('lodash'),
    moment: require('moment'),
  ```
5. Spacers are missing from layout. Fix.

## DB – Info
Three Mongo collections are created.
1.	LockManager – Contains user and schedule information
2.	LockSlots – Maps physical door lock slots to users
3.	LockLog – Logs events
Step 6. Add crypto-js
1.	Install npm modules.
-	npm i --save crypto-js lodash (if using Docker, bash and install (docker exec -it node-red-ui bash)
-	Update settings.js. Under functionGlobalContext add cryptojs: require(‘crypto-js’), lodash: require(‘lodash’)
-	Restart node-red
