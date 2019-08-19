# lock-manager
z-wave lock manager for home assistant built on node-red using docker.

## Purpose
Home Assistant doesn't have integrated lock management. I was using a paid app from RBoy when I was managing it using SmartThings. I've seen (and experimented) with *pure* Home Assistant implementations but I don't like using Home Assistant for state management and the input variables required to make it work created a lot of noise. Also, YAML is not intuitive to try and write functionality for me. When I started this I thought it would be simple and a couple hours of work. It's grown and grown.
# Installation
I'm using Docker and Docker Compose. It's not required but the instructions are written for it.

Here is a sample docker-compose.yml fragment. 

```
version: '3.6'
services:
  lock-manager:
    container_name: lock-manager
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

1. Clone the repo
```
git clone git@github.com:jayheavner/lock-manager.git
```
2. Navigate to the directory
   ```
   cd lock-manager
   ```

3. Install dependent packages
   ```
   npm i --save

4. Copy the settings.js.example file to settings.js
   ```
   sudo cp settings.js.example settings.js
   ```

5. Edit the settings.js file

   Three sections need to be changed *this is technically optional but highly recommended*
   - credentialSecret - Key that node-red uses to encrypt. If not specified it will create one and store in
   .config.json but better to create your own. You can use any random string.
   - adminAuth - Secures the node-red backend. Add the username and generate a hashed password. See the [node-red documentation](https://nodered.org/docs/user-guide/runtime/securing-node-red) for more info.
   - httpNodeAuth - Secures the /ui interface. See the documentation.
   
6. Bring up the container.
   ```
   docker-compose up -d
   ```
7. Navigate to the instance and finish setup.
   ```
   http://<ip>:1880
   ```
   - Add salt for encryption.
     - Click the encryption tab.
     - Double-click the *Put encryption key in global context* node
     - Enter the salt value in the first box
     - Click *Done*
    
   **_DON'T LOSE THIS VALUE OR YOU WILL NOT BE ABLE TO DECRYPT YOUR DATA!!!. STORE IT SOMEWHERE SAFE OR USE A VALUE YOU CAN REMEMBER._**

   - Update the Home Assistant configuration node.
     - Open the *Configuration nodes*
     - Find and double-click the *Home Assistant* node
     - Enter your home assistant url and Access Token. To generate a long-lived access token,. go to your profile in Home Assistant, scroll to the botton, and click the *Create token* under the *Long-Lived Access Tokens* section.
     - Click *Done*
     
   *Sometimes these nodes need to be opened and closed and redeployed at least once to work. I don't know why that is. If you see
   errors in node-red, try this.*

8. Change the various entity_ids to match your lock's entity_id. there are a couple places on the Event Handler tab. I think everything else uses the node_id, which is defined in global vars.

10. Deploy

11. Open the web ui interface
   ```
   http://<ip>:1880/ui
   ```

## DB – Info
Three Mongo collections are created.
1.	LockUsers – Contains user and schedule information
2.	LockSlots – Maps physical door lock slots to users
3.	LockLog – Logs events

## Variables
There is a node named 'Variables' that sets all the variables used. Eventually, I will move some of this into a settings page. 

```This sets all variables that would otherwise be 
hard coded into global/flow for use.

use_keypad - (boolean) true uses the keypad, false does not.

passcode_value - the number that unlocks with the pin pad (global)

passcode_timeout - number of seconds until passcode is required.

lock_node_id - the node_id of the door lock in question

code_length - a number between 4 and whatever that corresponds to the length of the code.
**For Schlage, all codes have to be the same length.**
NOTE: This does not change the lock settings, only specifies what's accepted in the form.

max_supported_slots - the number of slots the lock supports.

slot_offset - if there are slots that should never be touched, say slots 1-3, then enter a slot_offset
value of 3. Default is 0.

slot_offset_users - JSON array of users to handle the slot offsets. An example with 1 slot offset might be
`[{ "slot": 0, "name": "Lock button" }, { "slot": 1, "name": "Family" }]`

--why is this here? During testing I didn't want to mess with the important codes my family uses day-to-day.

devices_for_home - array of entity_id to check for home/away. 
Any device home is considered some_home, all devices home is considered all_home, all devices away is considered all_away.
This is for 'allow when home' checks but could be expanded.

notify_devices - JSON array of devices to receive notifications. Example: ['notify.ios_my_iphone']

notify_on_lock - uses the notify device and sends a notification to that device when door is locked from the keypad.

notify_on_manual_lock - uses the notify device and sends a notification to that device when door is locked from the inside or with a key.

notify_on_unlock - uses the notify device and sends a notification to that device when door is unlocked from the keypad.

notify_on_manual_unlock - uses the notify device and sends a notification to that device when door is unlocked from the inside or with a key.

use_encryption - specifies whether or not to encrypt the codes in the db.

disable_all_codes_when_asleep - clears all codes, except slot codes, while sleeping. 

use_clearcode - uses the lock.clear_usercode. This is **NOT** supported in Home Assistant without some work. System will always overwrite
codes with a random code but if this is true it will take the extra step of trying to clear the lock slot.

STILL IN PROGRESS
log_manual_lock - writes manual lock activities to log.

log_manual_unlock - writes manual lock activities to log.

log_user_changes - logs create, update, delete activities. *Note: masks code if use_encryption is true*

log_scheduled_changes - logs activities based on schedule. Also, logs one-time code usage.

keep_log_days - number of days to keep log data. Older data is removed.
```

## Security
Given that this tool manages locks which gain access to one's home, security is important. I strongly recommend using all of the built in security features. There are two additional security features that can be enabled through the settings.
  - use_encrytion - Encrypts data in the LockUsers table so if someone gains access to your database they won't be able to see users and codes.
  - use_keypad - Enforces additional security by requiring a pin before gaining access.
  
## Other
I am using a second instance of node-red for the lock manager. I used node-red for a lot of stuff and I have several instances running. If you don't have the option (or desire) to run multiple instances, you could simplify a lot of it by stripping out the encryption, db, and logging and just importing the flow. I plan to build a stripped-down version more appropriate for importing into an existing node-red instance.

## TO-DO
 - [ ] Truncate loggging
 - [ ] onfirm all logging flags are logging
 - [ ] Disable overnight flag
 - [ ] Disable on vacation flag
 - [ ] Update immediately on Save (don't wait for schedule to pick up changes)
 - [ ] Handle moving from decrypted to encrypted and back
