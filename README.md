# push-notification-with-angularjs-and-cordova

### In this tutorial you will learn How to setup Push Notification Server for Apple and Google Cloud

# Install required cordova plugins
We will now install required cordova plugins to make push notifications work.

```sh
 cordova plugin add https://github.com/Garib-ka-code/phonegap-build-PushPlugin.git
 cordova plugin add cordova-plugin-device
 cordova plugin add cordova-plugin-media
 cordova plugin add cordova-plugin-vibration 
```

### Include Push Angular Service

```js
  app.factory('pushService', function($q, $window) {

  var pushConfig = {};
  if (device.platform == 'android' || device.platform == 'Android') {
    pushConfig = {
      "senderID":"##########",
      "ecb":"onNotificationGCM"
    };
  } else {
    pushConfig = {
      "badge":"true",
      "sound":"true",
      "alert":"true",
      "ecb":"onNotificationAPN"
    };
  }
  
  // handle GCM notifications for Android
  $window.onNotificationGCM = function (event) {
    switch (event.event) {
      case 'registered':
        if (event.regid.length > 0) {
          // Your GCM push server needs to know the regID before it can push to this device
          // here is where you might want to send it the regID for later use.
          console.log("regID = " + event.regid);
          console.log("userID = " + window.localStorage.getItem("userID"));
          console.log("UserType = "+ window.localStorage.getItem("userType"));
          //send device reg id to server
        }
        break;

      case 'message':
          // if this flag is set, this notification happened while we were in the foreground.
          // you might want to play a sound to get the user's attention, throw up a dialog, etc.
          if (event.foreground) {
            console.log('INLINE NOTIFICATION');
            var my_media = new Media("/android_asset/www/" + event.soundname);
            my_media.play();
          } else {
            if (event.coldstart) {
                console.log('COLDSTART NOTIFICATION');
            } else {
                console.log('BACKGROUND NOTIFICATION');
            }
          }

          navigator.notification.alert(event.payload.message);
          console.log('MESSAGE -> MSG: ' + event.payload.message);
          //Only works for GCM
          console.log('MESSAGE -> MSGCNT: ' + event.payload.msgcnt);
          //Only works on Amazon Fire OS
          console.log('MESSAGE -> TIME: ' + event.payload.timeStamp);
          break;

      case 'error':
          console.log('ERROR -> MSG:' + event.msg);
          break;

      default:
          console.log('EVENT -> Unknown, an event was received and we do not know what it is');
          break;
    }
  };

  // handle APNS notifications for iOS
  $window.successIosHandler = function (result) {
    console.log('result = ' + result);
  };

  $window.onNotificationAPN = function (e) {
    if (e.alert) {
      console.log('push-notification: ' + e.alert);
      navigator.notification.alert(e.alert);
    }

    if (e.sound) {
      var snd = new Media(e.sound);
      snd.play();
    }

    if (e.badge) {
      pushNotification.setApplicationIconBadgeNumber("successIosHandler", e.badge);
    }
  };
  
  return {
    register: function () {
      var q = $q.defer();
      
      window.plugins.pushNotification.register(
      function (result) {
          q.resolve(result);
      },
      function (error) {
          q.reject(error);
      },
      pushConfig);
      
      return q.promise;
    }
  }
});
```

### Example of SenderId
https://phonegappro.com/tutorials/apache-cordova-phonegap-push-notification-tutorial-part-1/

### call factory angular service to main controller

```js
  pushService.register().then(function(result) {
      // Success! 
    }, function(err) {
        // An error occured. Show a message to the user
    });
```

### Sending Push Notifications
To send push notifications to devices who installed our app, (as we have device regIDs already stored in our database).

Upload PushServer php class to your server:

```php
<?php

/**
 * Usage:
 * $message = 'My First Push Notification!';
 * $pushServer = new PushSerer();
 * $pushServer->pushToGoogle('REG-ID-HERE', $message);
 * $pushServer->pushToApple('DEVICE-TOKEN-HERE', $message);
 */
class PushServer
{
    private $googleApiKey = 'YOUR-GOOGLE-API-KEY-HERE';
    private $googleApiUrl = 'https://android.googleapis.com/gcm/send';

    private $appleApiUrl = 'ssl://gateway.sandbox.push.apple.com:2195';
    private $privateKey = 'ck.pem';
    private $privateKeyPassPhrase = 'YOUR-PRIVATE-KEY-PASSPHRASE-HERE';

    public function pushToGoogle($regId, $message)
    {
        $fields = array(
            'registration_ids' => $regId,
            'data' => array("message" => $message),
        );
        $headers = array(
            'Authorization: key=' . $this->googleApiKey,
            'Content-Type: application/json'
        );

        // Open connection
        $ch = curl_init();

        // Set the url, number of POST vars, POST data
        curl_setopt($ch, CURLOPT_URL, $this->googleApiUrl);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($fields));

        // Execute post
        $result = curl_exec($ch);
        // Close connection
        curl_close($ch);

        return $result;
    }

    public function pushToApple($deviceToken, $message)
    {
        $ctx = stream_context_create();
        stream_context_set_option($ctx, 'ssl', 'local_cert', $this->privateKey);
        stream_context_set_option($ctx, 'ssl', 'passphrase', $this->privateKeyPassPhrase);

        // Open a connection to the APNS server
        $fp = stream_socket_client(
            $this->appleApiUrl,
            $err,
            $errstr,
            60,
            STREAM_CLIENT_CONNECT | STREAM_CLIENT_PERSISTENT,
            $ctx);

        if (!$fp)
            exit("Failed to connect: $err $errstr" . PHP_EOL);

        echo 'Connected to APNS' . PHP_EOL;

        // Create the payload body
        $body['aps'] = array(
            'alert' => $message,
            'sound' => 'default',
            'badge' => +1
        );

        // Encode the payload as JSON
        $payload = json_encode($body);

        // Build the binary notification
        $msg = chr(0) . pack('n', 32) . pack('H*', $deviceToken) . pack('n', strlen($payload)) . $payload;

        // Send it to the server
        $result = fwrite($fp, $msg, strlen($msg));

        // Close the connection to the server
        fclose($fp);

        return $result;
    }
}
```

That's all !!
