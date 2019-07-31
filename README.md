# lock-manager
z-wave lock manager for home assistant built on node-red using docker

# Installation
I'm using Docker and Docker Compose. It's not required. You could simply import the flow, add the palette packages, and make the changes to the settings.

Here is a sample docker-compose.yml file. Note: I'm actually spinning up two node-red instances; one is for web end-points and automations and the other is strictly for this ui. I'm changing the exposed port on the second instance. See the note about security for my rationale.

```version: '3.6'
services:
  node-red-ui:
    container_name: node-red-ui
    restart: always
    image: nodered/node-red-docker:v10
    user: 'root:root'
    ports:
      - '1880:1880'
    volumes:
      - ./node-red-ui/data:/data:Z
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    environment:
      - TZ=${TZ}
  
  mongodb:
    container_name: mongodb
    restart: unless-stopped
    image: mongo
    volumes:
      - ./mongo:/data/db
      - /etc/localtime:/etc/localtime:ro
    ports:
      - '27017:27017'
```

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
5. Install missing npm packages. This project is using crypto-js, moment, and lodash. To install, get to a bash prompt and navigate to the working folder and type `npm install --save`. It should pull from packages.json. If not, `npm install --save crypto-js moment lodash`.
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

## Variables
There is a node named 'Variables' that sets all the variables used. Eventually, I will move this into a settings page. There is a read me that describes variable usage. 

```This sets all variables that would otherwise be 
hard-coded into global/flow for use.

passcode - the number that unlocks with the pin pad (global)

lock_node_id - the node_id of the door lock in question

code_length - a number between 4 and whatever for codes.
**For Schlage, all codes have to be the same length.**

slot_offset - if there are slots that should never be touched, say slots 1-3, then enter a slot_offset
value of 3. Default is 0.

slot_offset_users - JSON array of users to handle the slot offsets. An example with 1 slot offset might be
`[{"slot":0,"name":"Lock button"},{"slot":1,"name":"Family"}]`


devices_for_home - array of entity_id to check for home/away.  [TODO]
Any device home is considered HOME. All devices aways is considered AWAY.

notify_devices - JSON array of devices to send notifications to. [TODO]

notify_on_lock - uses the notify device and sends a notification to that device when door is locked from the keypad. [TODO]

notify_on_manual_lock - uses the notify device and sends a notification to that device when door is locked from the inside or with a key. [TODO]

notify_on_unlock - uses the notify device and sends a notification to that device when door is unlocked from the keypad. [TODO]

notify_on_manual_unlock - uses the notify device and sends a notification to that device when door is unlocked from the inside or with a key. [TODO]

use_encryption - specifies whether or not to encrypt the codes in the db. [TODO]

disable_all_codes_when_asleep - clears all codes, except slot codes, while sleeping. [TODO]

use_clearcode - uses the lock.clear_usercode. This is **NOT** supported in Home Assistant without some work.
If value is false then a random number will be used to overwrite codes.

log_manual_lock - writes manual lock activities to log. [TODO]

log_manual_unlock - writes manual lock activities to log. [TODO]

log_user_created - writes user creation activities to log.

log_user_updated - writes user update activities to log.

log_user_deleted - writes user deletion activities to log.

log_scheduled_activity - writes any action taking based on a scheduled activity. Includes one-time use codes.

keep_log_days - number of days to keep log data. Older data is removed.
```
