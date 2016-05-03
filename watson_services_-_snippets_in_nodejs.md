# Watson Services - Snippets in NodeJS

## config\bluemix.js


```
'use strict';

/**
 * if VCAP_SERVICES exists then it returns
 * username, password and url
 * for the first service that stars with 'name' or {} otherwise
 * @param  String name, service name
 * @return {Object} the service credentials or {} if
 * name is not found in VCAP_SERVICES
 */
module.exports.getServiceCreds = function(name) {
    if (process.env.VCAP_SERVICES) {
        var services = JSON.parse(process.env.VCAP_SERVICES);
        for (var service_name in services) {
            if (service_name.indexOf(name) === 0) {
                var service = services[service_name][0];
                return {
                    url: service.credentials.url,
                    username: service.credentials.username,
                    password: service.credentials.password
                };
            }
        }
    }
    return {};
};
```

## config\error-handler.js
```
'use strict';

module.exports = function (app) {

  // catch 404 and forward to error handler
  app.use(function(req, res, next) {
    var err = new Error('Not Found');
    err.code = 404;
    err.message = 'Not Found';
    next(err);
  });

  // error handler
  app.use(function(err, req, res, next) {
    var error = {
      code: err.code || 500,
      error: err.error || err.message
    };
    console.log('error:', error);

    res.status(error.code).json(error);
  });

};
```


## config\express.js
```
'use strict';

// Module dependencies
var express    = require('express'),
  errorhandler = require('errorhandler'),
  bodyParser   = require('body-parser');

module.exports = function (app) {

  // Configure Express
  app.use(bodyParser.urlencoded({ extended: true }));
  app.use(bodyParser.json());

  // Setup static public directory
  app.use(express.static(__dirname + '/../public'));

  // Add error handling in dev
  if (!process.env.VCAP_SERVICES) {
    app.use(errorhandler());
  }

};
```

## app.js - init stub & error handling
```
'use strict';

var express  = require('express'),
  app        = express(),
  fs         = require('fs'),
  path       = require('path'),
  bluemix    = require('./config/bluemix'),
  extend     = require('util')._extend,
  watson     = require('watson-developer-cloud');

// Bootstrap application settings
require('./config/express')(app);

// if bluemix credentials exists, then override local
var dialogCredentials =  extend({
  url: 'https://gateway.watsonplatform.net/dialog/api',
  username: '<username>',
  password: '<password>',
  version: 'v1'
}, bluemix.getServiceCreds('dialog')); // VCAP_SERVICES

// if bluemix credentials exists, then override local
var nlcCredentials =  extend({
    "url": "https://gateway.watsonplatform.net/natural-language-classifier/api",
    username: "<username>",
    password: "<password>"
}, bluemix.getServiceCreds('natural_language_classifier')); // VCAP_SERVICES



var dialog_id_in_json = (function() {
  try {
    var dialogsFile = path.join(path.dirname(__filename), 'dialogs', 'dialog-id.json');
    var obj = JSON.parse(fs.readFileSync(dialogsFile));
    return obj[Object.keys(obj)[0]].id;
  } catch (e) {
  }
})();


var dialog_id = process.env.DIALOG_ID || dialog_id_in_json || '<missing-dialog-id>';

// Create the service wrapper
var dialog = watson.dialog(credentials);

...

// error-handler settings
require('./config/error-handler')(app);

var port = process.env.VCAP_APP_PORT || 3000;
app.listen(port);
console.log('listening at:', port);

```


## Dialog Service
<http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/dialog/api/v1/>

### List all the dialogs you have

```
var watson = require('watson-developer-cloud');

var dialog = watson.dialog({
  username: '<username>',
  password: '<password>',
  version: 'v1'
});

dialog.getDialogs({}, function (err, dialogs) {
  if (err)
    console.log('error:', err);
  else
    console.log(JSON.stringify(dialogs, null, 2));
});

```

### POST - Creates a dialog template for maintaining conversations. 

```
var fs     = require('fs');
var watson = require('watson-developer-cloud');
var dialog_service = watson.dialog({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

var params = {
  name: 'my-dialog',
  file: fs.createReadStream('template.xml')
};

dialog_service.createDialog(params, function(err, dialog) {
  if (err)
    console.log(err)
  else
    console.log(dialog);
});
```

### Deletes a dialog template and all associated data
```
var watson = require('watson-developer-cloud');
var dialog_service = watson.dialog({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

dialog_service.deleteDialog({dialog_id: '{dialog_id}'}, function(err, dialog) {
    if (err)
      console.log(err)
    else
      console.log(dialog);
});
```

### PUT - Update an existing dialog template
```
var fs     = require('fs');
var watson = require('watson-developer-cloud');
var dialog_service = watson.dialog({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

var params = {
  dialog_id: '{dialog_id}',
  file: fs.createReadStream('template.xml')
};

dialog_service.updateDialog(params, function(err, dialog) {
  if (err)
    console.log(err)
  else
    console.log(dialog);
});
```

### POST - Converse - Start a new conversation or continue an existing one.
```
var watson = require('watson-developer-cloud');
var dialog_service = watson.dialog({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

var params = {
  conversation_id: '{conversation_id}',
  dialog_id: '{dialog_id}',
  client_id: '{client_id}',
  input:     '{input}'
};

dialog_service.conversation(params, function(err, conversation) {
  if (err)
    console.log(err)
  else
    console.log(conversation);
});
```

### POST - Converse - exposed as a REST API method
```
app.post('/conversation', function(req, res, next) {
  var params = extend({ dialog_id: dialog_id }, req.body);
  dialog.conversation(params, function(err, results) {
    if (err)
      return next(err);
    else
      res.json({ dialog_id: dialog_id, conversation: results});
  });
});

```

### GET - Get a Conversation history


```
var watson = require('watson-developer-cloud');
var dialog_service = watson.dialog({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

var params = {
  dialog_id: '{dialog_id}',
  date_from: '{date_from}',
  date_to:   '{date_to}',
  limit:     '{limit}',
  offset:    '{offset}'
};

dialog_service.getConversation(params, function(err, session) {
  if (err)
    console.log(err)
  else
    console.log(session);
});
```

### GET - Get profile variables
```
var watson = require('watson-developer-cloud');
var dialog_service = watson.dialog({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

var params = {
  dialog_id: '{dialog_id}',
  client_id: {client_id},
  name: {name}
};

dialog_service.getProfile(params, function(err, profile) {
  if (err)
    console.log(err)
  else
    console.log(profile);
});
```

### GET - Get profile variables - exposed as REST API Method
```
app.post('/profile', function(req, res, next) {
  var params = extend({ dialog_id: dialog_id }, req.body);
  dialog.getProfile(params, function(err, results) {
    if (err)
      return next(err);
    else
      res.json(results);
  });
});
```

## Natural Language Classifier
<http://www.ibm.com/smarterplanet/us/en/ibmwatson/developercloud/natural-language-classifier/api/v1/>

### POST - Create Classifier
```
var watson = require('watson-developer-cloud');
var fs     = require('fs');

var natural_language_classifier = watson.natural_language_classifier({
  url: 'https://gateway.watsonplatform.net/natural-language-classifier/api',
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

var params = {
  language: 'en',
  name: 'My Classifier',
  training_data: fs.createReadStream('./train.csv')
};

natural_language_classifier.create(params, function(err, response) {
  if (err)
    console.log(err);
  else
    console.log(JSON.stringify(response, null, 2));
});
```

### GET - List classifiers
```
var watson = require('watson-developer-cloud');

var natural_language_classifier = watson.natural_language_classifier({
  url: 'https://gateway.watsonplatform.net/natural-language-classifier/api',
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

natural_language_classifier.list({},
  function(err, response) {
    if (err)
        console.log('error:', err);
      else
        console.log(JSON.stringify(response, null, 2));
  }
);
```

### GET - Retrieve status / get infomration about a classifier
```
var watson = require('watson-developer-cloud');

var natural_language_classifier = watson.natural_language_classifier({
  url: 'https://gateway.watsonplatform.net/natural-language-classifier/api',
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

natural_language_classifier.status({
  classifier_id: '10D41B-nlc-1' },
  function(err, response) {
    if (err)
      console.log('error:', err);
    else
      console.log(JSON.stringify(response, null, 2));
  }
);
```

### Delete a classifier
```
var watson = require('watson-developer-cloud'({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

natural_language_classifier.remove({
  classifier_id: '10D41B-nlc-1' },
  function(err, response) {
    if (err)
      console.log('error:', err);
    else
      console.log(JSON.stringify(response, null, 2));
});
```

### Classify
<h6> Returns label information for the input. The status must be "Available" before you can classify calls. Use the Get information about a classifier method to retrieve the status.</h6>

```
var watson = require('watson-developer-cloud')({
  username: '{username}',
  password: '{password}',
  version: 'v1'
});

natural_language_classifier.classify({
  text: 'How hot will it be today?',
  classifier_id: '10D41B-nlc-1' },
  function(err, response) {
    if (err)
      console.log('error:', err);
    else
      console.log(JSON.stringify(response, null, 2));
});
```

## Error Handler
### Error 404
```
// catch 404 and forward to error handler
app.use(function(req, res, next) {
    var err = new Error('Not Found');
    err.code = 404;
    err.message = 'Not Found';
    next(err);
});

```

### Error 500
```
// 500 error message
var errorMessage = 'There was a problem with the request, please try again';

// non 404 error handler
app.use(function(err, req, res) {
    var error = {
        code: err.code || 500,
        error: err.message || err.error || errorMessage
    };

    console.log('error: ' + JSON.stringify(error) + " for url: " + req.url);
    res.status(error.code).json(error);
});
```
