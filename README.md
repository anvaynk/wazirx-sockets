
# WazirX Socket Event Documentation

## Step 1: Install Pusher 
Pusher supports multiple libraries for various clients:
https://pusher.com/docs/channels/channels_libraries/libraries#client-libraries

## Step 2: Open a connection to Channels
There are two types of channels:
 1. Public channels
 2. Private channels

Create new instance of `Pusher` to open a connection to channels as follows:
```
var WAZIRX_PUSHER_KEY = "47bd0a9591a05c2a66db";
var pusher = new Pusher(WAZIRX_PUSHER_KEY, options);
```
Following is the structure of the `options` object:
```
{
	cluster: 'ap2', 
	encrypted: true,
	authorizer: (channel) => ({
		authorize: async (socketID, callback) => {
			try {
				const authData = await getPusherAuth(channel.name, socketID);
				callback(false, authData);
			} catch (e) {
				callback(true);
			}
		}
	});
}
```
This authorizer method is used to authenticate private channels. Any channel that starts with `private-` is a private channel.


### Implementing `getPusherAuth` method
	

The `getPusherAuth(channelName, socketId)` method is an asynchronous method that makes an API call to fetch 'auth' object that is needed for Pusher to authenticate the connection.

#### Base URL:
```
https://x.wazirx.com/pusher/auth
```

####  Method:

```
POST
```

####  Headers:

	'Content-Type': 'application/x-www-form-urlencoded'
	'access-key': **YOUR_ACCESS_KEY**
	'api-key': **API_KEY**
	'signature': a unique hash generated based on the payload and tonce
	'tonce': current date & time in milleseconds

#### Query Params:
	'socket_id': **SOCKET_ID** parameter passed to getPusherAuth method from Authorizer
	'channel_name': **CHANNEL_NAME** parameter passed to getPusherAuth method from Authorizer
________
***Sample cURL:***
```
curl 'https://x.wazirx.com/pusher/auth?socket_id=<SOCKET_ID>&channel_name=<CHANNEL_NAME>' \
  -X 'POST' \
  -H 'authority: x.wazirx.com' \
  -H 'content-length: 0' \
  -H 'access-key: <ACCESS_KEY>' \
  -H 'tonce: 1614779321863' \
  -H 'signature: <SIGNATURE>' \
  -H 'api-key: <API_KEY>' \
  -H 'user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 11_2_2) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/88.0.4324.192 Safari/537.36' \
  -H 'content-type: application/x-www-form-urlencoded' \
  --compressed
```
________
### How to generate signature:

Following inputs are needed for signature generation:
1. **ACCESS_KEY**: Access key of user account's active session 
2. **SECRET_KEY**: Secret Key of user account's active session
3. **BASE_URL**: Absolute URL of the API route.
4. **METHOD**: Method of API (GET | POST | PUT | DELETE)
5. **TONCE**: Current time in milliseconds that is also passed as a header in the API call.

**Step 1:**
Create a payload string as follows:

```
payload = `${METHOD}|access-key=${ACCESS_KEY}&tonce=${TONCE}|${BASE_URL}|`;
```

**Step 2:**
Create a HMAC-SHA256 authenticated hash by passing the `payload` as the input string and SECRET_KEY as the private key. 
The hex format of this hash is the signature.

***Sample Code***
```
import forge from 'node-forge';

ACCESS_KEY = ***YOUR_ACCESS_KEY***
SECRET_KEY = ***YOUR_SECRET_KEY***

const getSignature = (method, baseURL, tonce) => {
  const payload = `${method}|access-key=${ACCESS_KEY}&tonce=${tonce}|${baseURL}|`;
  const hmac = forge.hmac.create();
  hmac.start('sha256', SECRET_KEY);
  hmac.update(payload);
  return hmac.digest().toHex();
};
```

## Step 3: Subscribing to channels
Reference: https://pusher.com/docs/channels/getting_started/javascript#subscribe-to-a-channel
```
var channel = pusher.subscribe(<CHANNEL_NAME>);
```

## Step 4: Listening to events

Reference: 
https://pusher.com/docs/channels/getting_started/javascript#listen-for-events-on-your-channel

```
channel.bind(<EVENT_NAME>, function(data) {
  alert('An event was triggered with message: ' + data.message);
});
```

# LIST OF CHANNELS AND EVENTS:

***1. Global : `market-global`***
All the global events can be listened using this channel. This channel supports the following events:

List of Events:
 1. `tickers` :  Ticker data for all markets is available through this event. Note that this event will only contain data for those markets whose latest price is updated. To fetch data for all markets, please use the ticker api.
 

***2. Market : `market-<marketName>-global`***
Here, `marketName` is the name of the market whose events you want to listen.
For example: The market name for BTC-INR market is `btcinr`. So the channel name would be `market-btcinr-global`.

List of Events:
1. `update` : This event will be triggered every 3 seconds and will contain the order book.
2. `trades`: This event will be triggered every time a trade happens on the system for this selected market.

***3. Private Channel: `private-<sn>`***

This is a private channel and only authenticated users can access this channel. 
The `sn` is a unique identifier for each user in wazirx's system and can be accessed from the data of `members/me` api.
For Example, if the `sn` for a user is `WAZRXQWERTYUISAMPLE`, then the name of the channel would be `private-WAZRXQWERTYUISAMPLE`.
Data sent through the events on this channel is private and can only be accessed by authenticated users.

List of Events:

 1. `order`: This event is triggered when an order is placed/executed/cancelled.
 2. `account`: This event will be triggered when there is a change in the balance of funds in user's wallet.
