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

use_keypad - (boolean) true uses the keypad, false does not.

passcode_value - the number that unlocks with the pin pad (global)

passcode_timeout - number of seconds until passcode is required.

lock_node_id - the node_id of the door lock in question

code_length - a number between 4 and whatever that corresponds to the length of the code.
**For Schlage, all codes have to be the same length.**
NOTE: This does not change the lock settings, only specifies what's accepted in the form.

slot_offset - if there are slots that should never be touched, say slots 1-3, then enter a slot_offset
value of 3. Default is 0.

slot_offset_users - JSON array of users to handle the slot offsets. An example with 1 slot offset might be
`[{ "slot": 0, "name": "Lock button" }, { "slot": 1, "name": "Family" }]`

--why is this here? During testing I didn't want to mess with the important codes my family uses day-to-day.

devices_for_home - array of entity_id to check for home/away. 
Any device home is considered some_home, all devices home is considered all_home, all devices aways is considered all_away.
This is for 'allow when home' checks but could be expanded.

notify_devices - JSON array of devices to receive notifications. Example: ['notify.ios_my_iphone']

notify_on_lock - uses the notify device and sends a notification to that device when door is locked from the keypad.

notify_on_manual_lock - uses the notify device and sends a notification to that device when door is locked from the inside or with a key.

notify_on_unlock - uses the notify device and sends a notification to that device when door is unlocked from the keypad.

notify_on_manual_unlock - uses the notify device and sends a notification to that device when door is unlocked from the inside or with a key.

use_encryption - specifies whether or not to encrypt the codes in the db.

disable_all_codes_when_asleep - clears all codes, except slot codes, while sleeping. 

use_clearcode - uses the lock.clear_usercode.This is **NOT** supported in Home Assistant without some work. System will always overwrite
codes with a random code but if this is true it will take the extra step of trying to clear the lock slot.

log_manual_lock - writes manual lock activities to log.

log_manual_unlock - writes manual lock activities to log.

keep_log_days - number of days to keep log data. Older data is removed.
```
## Security
Node-red UI is not secure by default. It can be secured in the settings **but** that means any requests to node-red are secured. I've got end-points setup for Oauth callbacks that I didn't want to secure. That leaves two options.

1. Spin up another instance of node-red and secure the UI in that instance.
2. Build some ghetto hack to secure the front end.

I'm using the first option but I've included my *hack* which is a keypad that prevents usage until a code (defined in variables) is entered. The only problem with this is that it's server side which isn't terribly effective (if it's unlocked, it's unlocked for everyone until it relocks). There's a time-out (also defined in variables) that relocks the interface. TO-DO Look at better ways to provide alternate security.
