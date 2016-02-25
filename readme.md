# Channel Chat Code Along

## Introduction

We are going to build a chat web app (similar to Slack) using the _MEAN_ stack and the `angular-fullstack` _Yeoman_ generator.

![Screen Shot](https://raw.github.com/ATL-WDI-Exercises/channel-chat-code-along/master/images/channel-chat-app.png)

## Steps

### Step 0 - Install / Update Your Tools

0a. Run the following script to ensure that you have an up-to-date _NodeJS_ environment and the latest version of the `generator-angular-fullstack` module.

```bash
#!/bin/bash

nvm install 5.6.0
nvm alias default 5.6.0
nvm use default

echo "updating npm"
npm update -g

MODULES="
jasmine-node
express
express-generator
bower
grunt-cli
gulp-cli
yo
generator-gulp-angular
generator-angular-fullstack
babel@5.8.35
"

for MODULE in $MODULES
do
  echo "#################################"
  echo "Installing or Updating $MODULE"
  echo "#################################"
  npm install -g $MODULE
done
```

You can also check the version of the `generator-angular-fullstack` generator:

```bash
npm list -g generator-angular-fullstack
/Users/drmikeh/.nvm/versions/node/v5.6.0/lib
└── generator-angular-fullstack@3.3.0
```

### Step 1 - Generate the Project

1a. Create a new directory for this project and run the _Yeoman_ `angular-fullstack` generator.

    mkdir channel-chat
    cd channel-chat
    yo angular-fullstack --skip-install

         _-----_
        |       |
        |--(o)--|   .--------------------------.
       `---------´  |    Welcome to Yeoman,    |
        ( _´U`_ )   |   ladies and gentlemen!  |
        /___A___\   '__________________________'
         |  ~  |
       __'.___.'__
     ´   `  |° ´ Y `

    Out of the box I create an AngularJS app with an Express server.

    # Client

    ? What would you like to write scripts with? Babel
    ? What would you like to write markup with? HTML
    ? What would you like to write stylesheets with? Sass
    ? What Angular router would you like to use? uiRouter
    ? Would you like to include Bootstrap? Yes
    ? Would you like to include UI Bootstrap? Yes

    # Server

    ? What would you like to use for data modeling? Mongoose (MongoDB)
    ? Would you scaffold out an authentication boilerplate? Yes
    ? Would you like to include additional oAuth strategies?
    ? Would you like to use socket.io? Yes

    # Project

    ? Would you like to use Gulp or Grunt? Gulp
    ? What would you like to write tests with? Jasmine
    You're using the fantastic NgComponent generator.

    Initializing yo-rc.json configuration.

    ...

1b. Install the `npm` modules

```bash
npm install
```

1c. Update the version of Angular

Edit `bower.json` and update the _AngularJS_ version to `1.5.0` (search and replace) and then run:

```bash
bower install
```

1d. Add Additional Bower Dependencies

```bash
bower install --save angular-animate
bower install --save animate-css       # Note: use animate-css instead of animate.css
```

1e. Edit `client/app/app.js` and add the 'ngAnimate' module to our app module's dependencies:

```javascript
angular.module('channelChatApp', [
  'channelChatApp.auth',
  'channelChatApp.admin',
  'channelChatApp.constants',
  'ngCookies',
  'ngResource',
  'ngSanitize',
  'btford.socket-io',
  'ui.router',
  'ui.bootstrap',
  'validation.match',   // add trailing comma
  'ngAnimate'           // add this line
])
```

1f. Test it out

```bash
gulp build
gulp serve
gulp serve:dist
```

1g. Initialize a _GIT_ repo and commit all changes:

```bash
git init
git add -A
git commit -m "Created the project."
git tag step1
```

### Step 2 - Create a RESTful API Endpoint and Seed Data for Channels and Messages

In this step we will create a new RESTful API endpoint and some seed data for chat app's messages and channels.

2a. Use the `angular-fullstack` _endpoint_ subgenerator to create a new RESTful endpoint for messages:

```bash
yo angular-fullstack:endpoint message
```

Accept the default value for the url.

> NOTE: The `angular-fullstack` generator has a set of subgenerators that can help with scaffolding out server endpoint code or client code.

2b. Edit the file `server/api/message/message.model.js` and set the content to:

```javascript
'use strict';

var mongoose = require('bluebird').promisifyAll(require('mongoose'));

var MessageSchema = new mongoose.Schema({
  text: String,
  createdAt: Date,
  user: { type: mongoose.Schema.ObjectId, ref: 'User' },
  channelId: mongoose.Schema.ObjectId
});

export default mongoose.model('Message', MessageSchema);
```

2c. Add authentication to the Message routes.

Edit `server/api/message/index.js` and set the content to:

```javascript
'use strict';

var express = require('express');
var controller = require('./message.controller');
import * as auth from '../../auth/auth.service';

var router = express.Router();

router.get('/',       auth.isAuthenticated(), controller.index);
router.get('/:id',    auth.isAuthenticated(), controller.show);
router.post('/',      auth.isAuthenticated(), controller.create);
router.put('/:id',    auth.isAuthenticated(), controller.update);
router.patch('/:id',  auth.isAuthenticated(), controller.update);
router.delete('/:id', auth.isAuthenticated(), controller.destroy);

module.exports = router;
```

> NOTE: the `angular-fullstack` sub-generators are not generating the nice ES6 code that we got from the main generator (sigh). But it is okay to mix the old and the new - ES6 is fully backwards compatible.

2d. Edit `server/api/message/message.controller.js` and add the following:

```javascript
import Channel from '../channel/channel.model';     // add this line
```

And in the same file update the implementation of the create function to the following:

```javascript
// Creates a new Message in the DB
export function create(req, res) {
  if (!req.user) {
    return res.status(404).send('It looks like you aren\'t logged in, please try again.');
  }
  Channel.findByIdAsync(req.body.channelId)
  .then(function(channel) {
    if (!channel) {
      return res.status(404).send('Channel not found.');
    }
    var newMessage = channel.messages.create({
      text: req.body.text,
      user: req.user,
      createdAt: new Date()
    });
    channel.messages.push(newMessage);
    return channel.save();
  })
  .then(respondWithResult(res, 201))
  .catch(handleError(res));
}
```

2e. Use the `angular-fullstack` _endpoint_ subgenerator to create a new RESTful endpoint for channels:

```bash
yo angular-fullstack:endpoint channel
```

Accept the default value for the url.

2e. Edit the file `server/api/channel/channel.model.js` and set the schema to:

```javascript
'use strict';

var mongoose = require('bluebird').promisifyAll(require('mongoose'));
var Message = require('../message/message.model');

var ChannelSchema = new mongoose.Schema({
  name: String,
  description: String,
  active: Boolean,
  owner: { type: mongoose.Schema.ObjectId, ref: 'User' },
  messages: [Message.schema]
});

export default mongoose.model('Channel', ChannelSchema);
```

2f. Edit the file `server/api/channel/channel.controller.js` and change each of the Channel.find Async calls to Channel.find calls and add the call to `populate` on each of them:

```javascript
...
-  Channel.findAsync()
+  Channel.find().populate('owner messages.user')
...
-  Channel.findByIdAsync(req.params.id)
+  Channel.findById(req.params.id).populate('owner messages.user')
...
-  Channel.findByIdAsync(req.params.id)
+  Channel.findById(req.params.id).populate('owner messages.user')
...
-  Channel.findByIdAsync(req.params.id)
+  Channel.findById(req.params.id).populate('owner messages.user')
```


2g. Add authentication to the Channel routes.

Edit `server/api/channel/index.js` and set the content to:

```javascript
'use strict';

var express = require('express');
var controller = require('./channel.controller');
import * as auth from '../../auth/auth.service';

var router = express.Router();

router.get('/',       auth.isAuthenticated(), controller.index);
router.get('/:id',    auth.isAuthenticated(), controller.show);
router.post('/',      auth.isAuthenticated(), controller.create);
router.put('/:id',    auth.isAuthenticated(), controller.update);
router.patch('/:id',  auth.isAuthenticated(), controller.update);
router.delete('/:id', auth.isAuthenticated(), controller.destroy);

module.exports = router;
```

2h. Set the content of `server/config/seed.js` to the following:

```javascript
/**
 * Populate DB with sample data on server start
 * to disable, edit config/environment/index.js, and set `seedDB: false`
 */

'use strict';
import Thing from '../api/thing/thing.model';
import User from '../api/user/user.model';
import Channel from '../api/channel/channel.model';

Thing.find({}).removeAsync()
  .then(() => {
    return Thing.create({
      name: 'Development Tools',
      info: 'Integration with popular tools such as Bower, Grunt, Babel, Karma, ' +
             'Mocha, JSHint, Node Inspector, Livereload, Protractor, Jade, ' +
             'Stylus, Sass, and Less.'
    }, {
      name: 'Server and Client integration',
      info: 'Built with a powerful and fun stack: MongoDB, Express, ' +
             'AngularJS, and Node.'
    }, {
      name: 'Smart Build System',
      info: 'Build system ignores `spec` files, allowing you to keep ' +
             'tests alongside code. Automatic injection of scripts and ' +
             'styles into your index.html'
    }, {
      name: 'Modular Structure',
      info: 'Best practice client and server structures allow for more ' +
             'code reusability and maximum scalability'
    }, {
      name: 'Optimized Build',
      info: 'Build process packs up your templates as a single JavaScript ' +
             'payload, minifies your scripts/css/images, and rewrites asset ' +
             'names for caching.'
    }, {
      name: 'Deployment Ready',
      info: 'Easily deploy your app to Heroku or Openshift with the heroku ' +
             'and openshift subgenerators'
    });
  }, handleError);

User.find({}).removeAsync()
  .then(() => {
    return User.createAsync({
      provider: 'local',
      name: 'Test User',
      email: 'test@example.com',
      password: 'test'
    }, {
      provider: 'local',
      role: 'admin',
      name: 'Admin',
      email: 'admin@example.com',
      password: 'admin'
    })
    .then(() => {
      console.log('finished populating users');
      return User.find();
    })
    .then((users) => {
      return createChannels(users);
    })
    .then((channels) => {
      console.log('created the following channels:\n', channels);
    }, handleError);
  });

function createChannels(users) {
  return Channel.find().remove({})
  .then(() => {
    let now = new Date();
    return Channel.create([
      {
        name: 'Classroom',
        description: 'Classroom discussion',
        active: true,
        owner: users[0]._id,
        messages: [
          { text: 'First message.',  createdAt: now, user: users[0]._id },
          { text: 'Second message.', createdAt: now, user: users[1]._id }
        ]
      },
      {
        name: 'Outcomes',
        description: 'I Need a job!',
        active: true,
        owner: users[0]._id,
        messages: [
          { text: 'Third message.',  createdAt: now, user: users[0]._id },
          { text: 'Fourth message.', createdAt: now, user: users[1]._id }
        ]
      },
      {
        name: 'Resources',
        description: 'Where can I get more info?',
        active: true,
        owner: users[1]._id,
        messages: [
          { text: 'Fifth message.', createdAt: now, user: users[0]._id },
          { text: 'Sixth message.', createdAt: now, user: users[1]._id }
        ]
      }
    ]);
  }, handleError);
}

function handleError(err) {
  console.log('ERROR', err);
}
```

> NOTE: If your `gulp serve` session is still running, it will reseed the database each time you make changes to any of the server files.

2i. Test out the RESTful endpoints

First you will need to disable AuthN/AuthZ in the `server/api/channel/index.js` file in order to test this route without a session. Just comment out or remove the calls to `auth.isAuthenticated()` but remember to put them back in after testing.

Then make sure you have `gulp serve` running and then use `httpie` or your browser to hit the following endpoints.

```
http localhost:9000/api/channels
http localhost:9000/api/channels/56cb661b4170468e361ca48a    # use any id from the above request
```

> Note that our _RESTful_ endpoints live under the path `/api` which makes it easy to mange requests for static assets (files) vs. requests for data (_RESTful_ endpoints).

2j. Commit your work

```bash
git add -A
git commit -m "Created a RESTful API Endpoint and Seed Data for Messages and Channels."
git tag step2
```

### Step 3 - Create a New Client Route for Channels

3a. Use the `angular-fullstack` _route_ subgenerator to create a new client route for our channels view:

```bash
yo angular-fullstack:route channels
```

You can accept the default values for the route.

3b. Modify the generated controller and view to use the `controllerAs` binding:

Edit `client/app/channels/channels.js` and add the `controllerAs` binding:

```javascript
'use strict';

angular.module('channelChatApp')
  .config(function ($stateProvider) {
    $stateProvider
      .state('channels', {
        url: '/channels',
        templateUrl: 'app/channels/channels.html',
        controller: 'ChannelsCtrl',                     // add trailing comma
        controllerAs: 'vm'                              // add this line
      });
  });
```

3c. Implement the Channels Controller and View:

Edit `client/app/channels/channels.controller.js` and set the content to:

```javascript
'use strict';

angular.module('channelChatApp')
  .controller('ChannelsCtrl', function(channelService) {
    var vm = this;
    vm.newMessage = 'type new message here';

    channelService.getChannels().then(function(response) {
      vm.channels = response.data;
      vm.selectedChannel = vm.channels.length > 0 ? vm.channels[0] : null;

    });

    vm.setSelected = function(channel) {
      vm.selectedChannel = channel;
    };

    vm.isSelected = function(channel) {
      return channel._id === vm.selectedChannel._id;
    };

    vm.sendMessage = function() {
      channelService.sendMessage(vm.newMessage, vm.selectedChannel)
      .then(function(response) {
        vm.newMessage = 'type new message here';
      });
    };
  });
```

Edit `client/app/channels/channels.html` and set the content to:

```html
<section class="container">
  <div class="row">
    <article class="col-md-6">
      <h3>Channels:</h3>
      <div class="panel panel-primary channel"
           ng-repeat="channel in vm.channels | orderBy: 'name'"
           ng-click="vm.setSelected(channel)">
        <div class="panel-heading">
          <h3 class="panel-title">
            <span ng-show="vm.isSelected(channel)" class="glyphicon glyphicon-ok" aria-hidden="true"></span> {{ channel.name }}
          </h3>
        </div>
        <div class="panel-body">
          <dl class="dl-horizontal">
            <dt>Description</dt><dd>{{ channel.description }}</dd>
            <dt>Active</dt><dd>{{ channel.active }}</dd>
            <dt>Owner</dt><dd>{{ channel.owner.name }}</dd>
            <dt>Messages</dt><dd>{{ channel.messages.length }}</dd>
          </dl>
        </div>
      </div>
    </article>
    <article class="col-md-6" style="height: 400px; overflow-y: scroll;">
      <h3>Channel {{ vm.selectedChannel.name }}</h3>
      <form>
        <textarea class="form-control"
                  rows="3"
                  ng-model="vm.newMessage"
                  tabindex="1"
                  onfocus="if (this.value=='type new message here') this.value = ''">
        </textarea>
        <br>
        <button type="submit" class="btn btn-primary" ng-click="vm.sendMessage()">Send</button>
      </form>
      <hr>
      <div class="animate-messages" ng-repeat="message in vm.selectedChannel.messages | orderBy: '-createdAt'">
        <dl>
          <dt><span class="text-primary"><strong>{{ message.user.name }}<strong></span> <small>{{ message.createdAt | date:"MM/dd/yyyy 'at' h:mma" }}</small></dt>
          <dd><div class="well" style="white-space: pre-wrap; padding: 9px;">{{ message.text }}</div></dd>
        </dl>
      </div>
    </article>
  </div>
</section>
```

#### Observations:
* We are using the Twitter Bootstrap grid to layout the Channel list and Message list side-by-side
* We use ng-repeat with the `orderBy` filter to get a nice sorted list of channels and messages. Messages are sorted in reverse chronological order of the `createdAt` date via the expression `| orderBy: '-createdAt'"`.
* We put an `ng-click` on the channel panels so that we can click and select the channel we want to work in.

3d. Create the `channelService`

Use the Yeoman generator to create an _Angular_ `channelService`:

```bash
yo angular-fullstack:service channelService
mv client/app/channelService/channelService.service.* client/app/channels/
rmdir client/app/channelService
```

> NOTE: I don't like the way the gnerator puts the serivce inside it's own folder so I added the `mv` and `rmdir` statements above.

Now add the following content to `client/app/channels/channelService.service.js`:

```javascript
'use strict';

angular.module('channelChatApp')
  .service('channelService', function($http) {

    var svc = this;
    svc.channels = [];

    svc.getChannels = function() {
      var promise= $http.get('/api/channels');
      promise.then(function(response) {
        svc.channels = response.data;
      });
      return promise;
    };

    svc.sendMessage = function(newMessage, channel) {
      return $http.post('/api/messages',
                        { text: newMessage,
                          channelId: channel._id
                        });
    };

    svc.findById = function(id) {
      return _.find(svc.channels, function(channel) {
        return channel._id === id;
      });
    };
  });
```

#### Observations
* We are using `$http` to call the server.
* We return a promise back to the controller.
* We cache up the results so that we can look up channels later.


3e. Add the _SCSS_ styling

Edit `client/app/channels/channels.scss` and add the following content:

```css
.channel {
  background: #eee;
  cursor: pointer;
  cursor: hand;
}

.animate-messages {
  &.ng-enter {
    animation: fadeInRight 0.3s;
  }
  &.ng-leave {
    animation: fadeOutLeft 0.3s;
  }
}
```

3f. Add a Channels link to the _NavBar_

Edit `client/components/navbar/navbar.html` and add the following line just after the `Admin` link:

```html
<li ng-show="nav.isLoggedIn()" ui-sref-active="active"><a ui-sref="channels">Channels</a></li>
```

3g. Test your work

> Before testing, restart your `gulp serve` session. It is a good idea to restart it after scaffolding so that all of the gulp injectors have a chance to inject the new files.

* Sign in using one of the seeded user accounts
* Navigate to the _Channels_ view and notice the seeded messages.
* Click on the different Channels
* Add some new messages.

> NOTE: For now you will have to refresh your page to see the new Messages. We will fix that in Step 4.

3h. Commit your work

```bash
git add -A
git commit -m "Created the client route, controller, and view for Channels."
git tag step3
```

### Step 4 - Add Live Updates to Channel Messages

To really make this chat app great we need to add the code for live updates whenever we or someone else posts a new message. Fortunately the `socket.io` library and _AngularJS_ make this very easy.

First we need to emit messages from the server whenver a new message is added to a channel.
Then we need to listen for those messages on the client-side and update our messages accordingly.

4a. Remove the registeration of the `Message` Model events

Currently we actually have _too many_ socket.io events occurring for our messages. Because we we chose to make our messages an embedded document of our Channel model, we only need to emit a message when we get a new one in the controller, not each time we save a Channel.

Edit `server/api/message/message.events.js` and comment out line indicated below:

```javascript
// Register the event emitter to the model events
for (var e in events) {
  var event = events[e];
  // Message.schema.post(e, emitEvent(event));        // comment out this line
}
```

4b. Emit a message event from the server-side Message Controller

Edit `server/api/message/message.controller.js` and make the following changes:

```javascript
var MessageEvents = require('./message.events');      // add this line

...

channel.messages.push(newMessage);
MessageEvents.emit('save', { message: newMessage, channelId: channel._id });  // add this line
return channel.save();
```

4c. Listen for the _New Message_ event in the client code

Edit `client/app/channels/channels.controller.js` and set the content to the following:

```javascript
'use strict';

angular.module('channelChatApp')
  .controller('ChannelsCtrl', function(channelService, $scope, socketFactory) {
    var vm = this;
    vm.newMessage = 'type new message here';

    // socket.io now auto-configures its connection when we ommit a connection url
    var ioSocket = io('', {
      // Send auth token on connection, you will need to DI the Auth service above
      // 'query': 'token=' + Auth.getToken()
      path: '/socket.io-client'
    });

    var socket = socketFactory({ ioSocket });

    channelService.getChannels().then(function(response) {
      vm.channels = response.data;
      vm.selectedChannel = vm.channels.length > 0 ? vm.channels[0] : null;

      // socket.syncUpdates('message', vm.selectedChannel.messages);
      // TODO: I need to handle messages that arrive on other channels.
      socket.on('message:save', function(eventData) {
        var message = eventData.message;
        var channelId = eventData.channelId;
        var affectedChannel = channelService.findById(channelId);
        var oldMessage = _.find(affectedChannel.messages, {_id: message._id});
        var index = affectedChannel.messages.indexOf(oldMessage);

        // replace oldMessage if it exists
        // otherwise just add message to the collection
        if (oldMessage) {
          affectedChannel.messages.splice(index, 1, message);
        } else {
          affectedChannel.messages.push(message);
        }
      });
    });

    $scope.$on('$destroy', function() {
      socket.unsyncUpdates('message');
    });

    vm.setSelected = function(channel) {
      vm.selectedChannel = channel;
    };

    vm.isSelected = function(channel) {
      return channel._id === vm.selectedChannel._id;
    };

    vm.sendMessage = function() {
      channelService.sendMessage(vm.newMessage, vm.selectedChannel)
      .then(function(response) {
        vm.newMessage = 'type new message here';
      });
    };
  });
```

4d. Test it out

* Sign in using one of the seeded user accounts
* Navigate to the _Channels_ view and notice the seeded messages.
* Click on the different Channels
* Add some new messages.

Now you should see that the live updates are working. You can even try it out in 2 browsers. Just log into the app with Chrome using the `admin` account and log into the app with Safari using the `test` account and try going back and forth adding messages and viewing the live updates!!!

4e. Commit your work

```bash
git add -A
git commit -m "Added support for live updates of new messages."
git tag step4
```

### Step 5 - Add Animation For Change in Message Count

We want to see a visual cue for when new messages appear in channels other than the one we are viewing. We will use some CSS animation and a custom AngularJS directive to mange the CSS class changes when the `channel.messages.length` value changes.

5a. Create the `animate-on-change` custom AngularJS directive.

```bash
mkdir client/components/animate-on-change
touch client/components/animate-on-change/animate-on-change.directive.js
```

Edit `client/components/animate-on-change/animate-on-change.directive.js` and add the following content:

```javascript
(function () {
  'use strict';

  angular.module('channelChatApp')
  .directive('animateOnChange', function($timeout) {
    return function(scope, element, attr) {
      scope.$watch(attr.animateOnChange, function(nv,ov) {
        if (nv !== ov) {
          element.addClass('changed');
          $timeout(function() {
            element.removeClass('changed');
          }, 1000);
        }
      });
    };
  });
})();
```

5b. Add the CSS for the animation.

Edit `client/app/channels/channels.scss` and add the following:

```css
[animate-on-change] {
  transition: all 0.5s;
  -webkit-transition: all 0.5s;
  &.changed {
    animation: shake 0.5s;
    background-color: red;
    transition: none;
    -webkit-transition: none;
  }
}
```

5c. Edit `client/index.html` and set the channel messages length display to the following:

```html
<dt>Messages</dt><dd><span class="badge" animate-on-change='channel.messages.length'>{{ channel.messages.length }}</span></dd>
```

5d. Test it out

Add some messages and see if the animations work.

5e. Commit your work

```bash
git add -A
git commit -m "Added animation for changes in number of channel messages."
git tag step5
```

## TODOS
* User can add new channels
  - add live updates for new Channels.
* User can subscribe / unsubsribe to a channel
