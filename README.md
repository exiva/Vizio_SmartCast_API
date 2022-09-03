# Vizio SmartCast API (2016+ Models)

[[_TOC_]]

## API Clients

The following API clients are available based on these API docs:

- [vizio-smart-cast](https://github.com/heathbar/vizio-smart-cast) (JavaScript)
- [pyvizio](https://github.com/vkorn/pyvizio) (Python)

## Overview
* On firmware versions older than 4.0 the API server runs on port `9000`, on 4.0 and newer the API server runs on port `7345`. Both are using https and will not respond to http.

	*Don't port forward it. There are some commands that can be executed
without authentication.*
* Certificate's CN is `BG2.prod.vizio.com` so will likely fail SSL validation.
* API includes a `Status` object, `URI` requested, and Execution `time` on every response. It's included below, but excluded in all of the examples for redundancy.

	```
	{
		...
		"STATUS": {
			"RESULT": String,
			"DETAIL": String
		},
		"URI": String,
		"TIME": String
	}
	```
	API does not seem to be fully restful, and doesn't apply proper status codes in responses. Verify request's result from the `status` object. Result codes included where applicable.
* All requests made should contain a JSON body with the header `Content-Type: application/json` set.
* When authentication is required, send `Auth: AUTH_TOKEN` header with request.
* This does not cover any MyVizio Account APIs.
* See an issue? Have a question? Open an issue, or find me on twitter [@exiva](https://twitter.com/exiva)

## Display Discovery

### Find IP Address
Perform an [SSDP query](https://sdkdocs.roku.com/display/sdkdoc/External+Control+API#ExternalControlAPI-SSDP(SimpleServiceDiscoveryProtocol)) for `ST: urn:schemas-kinoma-com:device:shell:1`

#### Example
```
M-SEARCH * HTTP/1.1
HOST: 239.255.255.250:1900
MAN: "ssdp:discover"
MX: 1
ST: urn:schemas-kinoma-com:device:shell:1
```

## Pairing
***required to control set.***
### Start Pairing
`PUT /pairing/start`

#### Body
```
{
	"DEVICE_NAME": String,
	"DEVICE_ID": String
}
```

#### Response
```
{
	"ITEM": {
		"PAIRING_REQ_TOKEN": Integer,
		"CHALLENGE_TYPE": Integer
	},
	...
}
```

#### cURL Example
`curl -k -H "Content-Type: application/json" -X PUT -d '{"DEVICE_ID":"12345","DEVICE_NAME":"cURL Example"}' https://myVizioTV:9000/pairing/start`

#### Notes
Save `DEVICE_ID`, you'll need it for the challenge or to cancel pairing.

### Submit Challenge
`PUT /pairing/pair`

#### Body
```
{
  "DEVICE_ID": String,
  "CHALLENGE_TYPE": Integer,
  "RESPONSE_VALUE": String,
  "PAIRING_REQ_TOKEN": Integer
}
```

#### Response
```
{
	"ITEM": {
		"AUTH_TOKEN": String
	},
	...
}
```

#### cURL Example
`curl -k -H "Content-Type: application/json" -X PUT -d '{"DEVICE_ID": "12345","CHALLENGE_TYPE": 1,"RESPONSE_VALUE": "1234","PAIRING_REQ_TOKEN": 654321}' https://myVizioTV:9000/pairing/pair`

#### Notes
`RESPONSE_VALUE` key is the code displayed on the TV.\
`PAIRING_REQ_TOKEN` key is returned from the /pairing/start request.

### Cancel Pairing
`PUT /pairing/cancel`

#### Body
```
{
  "DEVICE_ID": String,
  "CHALLENGE_TYPE": Integer,
  "RESPONSE_VALUE": "1111",
  "PAIRING_REQ_TOKEN": Integer
}
```

#### Response
```
{
	"ITEM": {},
	...
}
```

#### cURL Example
`curl -k -H "Content-Type: application/json" -X PUT -d '{"DEVICE_ID":"12345","CHALLENGE_TYPE": 1,"RESPONSE_VALUE": "1111","PAIRING_REQ_TOKEN": 0}' https://myVizioTV:9000/pairing/cancel`

#### Notes
For cancellation, `RESPONSE_VALUE` should be hard-coded to "1111".

### Appendix
#### Status Results
| RESULT                   | Meaning                       |
| ------------------------ | ----------------------------- |
| INVALID_PARAMETER        | Malformed Request             |
| MAX\_CHALLENGES_EXCEEDED | Too many failed pair attempts |
| SUCCESS                  | Successfully Paired           |
| PAIRING_DENIED           | Incorrect Pin                 |
| VALUE\_OUT\_OF_RANGE     | Pin out of range              |
| CHALLENGE_INCORRECT      | Incorrect challenge           |
| BLOCKED                  | Pairing in progress already   |


## Remote Control Methods
### Turning Set on From Sleep
The HTTP API server turns off when the set is sleeping. Send a WoL magic packet to turn it on. *not supported when set runs in Eco mode.*

### Power State
*Authenticated*

`GET /state/device/power_mode`

#### Response
```
{
	"ITEMS": [{
		"TYPE": "T_VALUE_V1",
		"CNAME": "power_mode",
		"NAME": "Power Mode",
		"VALUE": 1
	}],
	"PARAMETERS": {
		"HASHONLY": "FALSE",
		"FLAT": "TRUE",
		"HELPTEXT": "FALSE"
	}
	...
}
```

#### cURL Example
`curl -k -H "Content-Type: application/json" -H "AUTH: 123A456B" -X GET https://myVizioTV:9000/state/device/power_mode`

### Send remote control button press
*Authenticated*

`PUT /key_command/`

#### Body
```
{
	"KEYLIST": [{
		"CODESET": Integer,
		"CODE": Integer,
		"ACTION": String
	}]
}
```

#### cURL Example
`curl -k -H "Content-Type: application/json" -H "AUTH: 123A456B" -X PUT -d '{"KEYLIST": [{"CODESET": 5,"CODE": 0,"ACTION":"KEYPRESS"}]}' https://myVizioTV:9000/key_command/`

#### Notes
You can string together long remote actions by adding to the `keylist` array.

### Appendix
| Action   |
| -------- |
| KEYDOWN  |
| KEYUP    |
| KEYPRESS |

| Event Name   | Codeset | Code |
| ------------ | :-----: | :--: |
| Volume Down  | 5       | 0    |
| Volume Up    | 5       | 1    |
| Mute Off     | 5       | 2    |
| Mute On      | 5       | 3    |
| Mute Toggle  | 5       | 4    |
| Cycle Input  | 7       | 1    |
| Channel Down | 8       | 0    |
| Channel Up   | 8       | 1    |
| Previous Ch  | 8       | 2    |
| Power Off    | 11      | 0    |
| Power On     | 11      | 1    |
| Power Toggle | 11      | 2    |

## Video Input Methods
### Get currently selected input
*Authenticated*

`GET /menu_native/dynamic/tv_settings/devices/current_input`

#### Response
```
{
	"ITEMS": [{
		"NAME": String,
		"CNAME": String,
		"TYPE": String,
		"VALUE": String,
		"ENABLED": Boolean,
		"HASHVAL": Integer
	}],
	"NAME": String,
	"HASHLIST": Array,
	"GROUP": String,
	"PARAMETERS": {
		"HASHONLY": String,
		"FLAT": String,
		"HELPTEXT": String
	},
	...
}
```

#### cURL Example
`curl -k -H "Content-Type: application/json" -H "AUTH: 123A456B" -X GET https://myVizioTV:9000/menu_native/dynamic/tv_settings/devices/current_input`

### Get list of inputs
*Authenticated*

`GET /menu_native/dynamic/tv_settings/devices/name_input`

#### Response
```
{
	"ITEMS": [{
		"NAME": String,
		"CNAME": String,
		"TYPE": String,
		"VALUE": {
			"NAME": String,
			"METADATA": String
		},
		"ENABLED": Boolean,
		"HASHVAL": Integer
	},
	...
	],
	"NAME": String,
	"HASHLIST": Array,
	"GROUP": String,
	"PARAMETERS": {
		"HASHONLY": String,
		"FLAT": String,
		"HELPTEXT": String
	},
	...
}
```

#### cURL Example
`curl -k -H "Content-Type: application/json" -H "AUTH: 123A456B" -X GET https://myVizioTV:9000/menu_native/dynamic/tv_settings/devices/name_input`

### Change Input
*Authenticated*

`PUT /menu_native/dynamic/tv_settings/devices/current_input`

#### Body
```
{
	"REQUEST": "MODIFY",
	"VALUE": String,
	"HASHVAL": Integer
}
```

##### Expected Values
| PUT `current_input` | From `name_input`   |
| ------------------- | ------------------- |
| VALUE               | ITEMS[x].NAME       |
| HASHVAL             | Current HashValue from a "get" of current_input  |

Note: The hashvalue is from the current input, not the list of inputs.

#### cURL Example
`curl -k -H "Content-Type: application/json" -H "AUTH: 123A456B" -X PUT -d '{"REQUEST": "MODIFY","VALUE": "HDMI-1","HASHVAL": 1384176329}' https://myVizioTV:9000/menu_native/dynamic/tv_settings/devices/current_input`


### Launch TV App
*Authenticated*

`PUT /app/launch`

#### Body
```
{
	"VALUE": {
		"MESSAGE": String,
		"NAME_SPACE": Integer,
		"APP_ID": String
	}
}
```
*See [App IDs](#app-ids) for Values*

## Display Settings
### Read Settings
*Authenticated*

`GET /menu_native/dynamic/tv_settings/SETTINGS_CNAME`
*(See [Settings CNAMEs](#settings-cnames) for `SETTINGS_CNAME` values)*

#### Response
```
{
	"ITEMS": [{
		"NAME": String,
		"CNAME": String,
		"TYPE": String,
		"VALUE": String,
		"ENABLED": Boolean,
		"HASHVAL": Int,
		"ELEMENTS": Array
	},
	...
	],
	"NAME": String,
	"HASHLIST": Array,
	"GROUP": String,
	"PARAMETERS": {
		"HASHONLY": String,
		"FLAT": String,
		"HELPTEXT": String
	},
	...
}
```

#### Notes
`ELEMENTS` array is conditional.

#### cURL Example
`curl -k -H "Content-Type: application/json" -H "AUTH: 123A456B" -X GET https://myVizioTV:9000/menu_native/dynamic/tv_settings/picture`

### Write Settings
*Authenticated*

**Warning** Writing an invalid value may have the potential to brick sets. Use the `TYPE` key from `SETTINGS_CNAME` `ITEMS` array to determine the data type to write.

`PUT /menu_native/dynamic/tv_settings/SETTINGS_CNAME/ITEMS_CNAME`

#### Body
```
{
	"REQUEST": "MODIFY",
	"HASHVAL": Integer,
	"VALUE": String/Integer/Boolean
}
```

#### Notes
Obtain `ITEMS_CNAME` and `HASHVAL` values from the `SETTINGS_CNAME` `ITEMS` array.

#### cURL Example
`curl -k -H "Content-Type: application/json" -H "AUTH: 123A456B" -X PUT -d '{"REQUEST": "MODIFY", "HASHVAL": 831638627, "VALUE": "On"}' https://myVizioTV:9000/menu_native/dynamic/tv_settings/picture/game_low_latency`

### Appendix
#### Settings CNAME's
| SETTINGS_CNAME                                 |
| ---------------------------------------------- |
| picture                                        |
| picture/picture_size                           |
| picture/picture_position                       |
| picture/picture_mode_edit                      |
| picture/color\_calibration                     |
| picture/color\_calibration/color_tuner         |
| picture/color\_calibration/calibration_tests   |
| audio                                          |
| timers                                         |
| network                                        |
| channels                                       |
| closed_captions                                |
| devices                                        |
| system                                         |
| system/system\_information                     |
| system/system\_information/tv_information      |
| system/system\_information/tuner_information   |
| system/system\_information/network_information |
| system/system\_information/uli_information     |
| mobile_devices                                 |
| cast                                           |

### App IDs
| Name                          | Name_Space | App_id   | Message  |
| ----------------------------- | :--------: | :------: | -------- |
| Hulu                          | 2          | 3        | None     |
| Prime Video                   | 2          | 4        | None     |
| Crackle                       | 2          | 5        | None     |
| iHeartRadio                   | 2          | 6        | None     |
| Fandango Now                  | 2          | 7        | None     |
| Plex                          | 2          | 9        | None     |
| NBC                           | 2          | 10       | None     |
| Baeble                        | 2          | 11       | None     |
| Curiosity Stream              | 2          | 12       | None     |
| Newsy                         | 2          | 15       | None     |
| Dove Channel                  | 2          | 16       | None     |
| ConTV                         | 2          | 18       | None     |
| Toon Goggles                  | 2          | 21       | None     |
| Vudu                          | 2          | 21       | https://my.vudu.com/castReceiver/index.html?launch-source=app-icon |
| WatchFree                     | 2          | 22       | None     |
| FilmRise                      | 2          | 24       | None     |
| TasteIt                       | 2          | 26       | None     |
| AsianCrush                    | 2          | 27       | https://html5.asiancrush.com/?ua=viziosmartcast |
| DAZN                          | 2          | 34       | None     |
| Flixfling                     | 2          | 36       | None     |
| CBS All Access                | 2          | 37       | None     |
| FitFusion by Jillian Michaels | 2          | 39       | None     |
| Redbox                        | 2          | 41       | None     |
| CBS CBS News                  | 2          | 42       | None     |
| Cocoro TV                     | 2          | 55       | None     |
| Love Destination              | 2          | 57       | None     |
| Haystack TV                   | 0          | 898AF734 | {"CAST_NAMESPACE":"urn:x-cast:com.google.cast.media","CAST_MESSAGE":{"type":"LOAD","media":{},"autoplay":true,"currentTime":0,"customData":{"platform":"sctv"}}} |
| XUMO                          | 0          | 36E1EA1F | {"CAST_NAMESPACE":"urn:x-cast:com.google.cast.media","CAST_MESSAGE":{"type":"LOAD","media":{},"autoplay":true,"currentTime":0,"customData":{}}} |
| Pluto TV                      | 0          | E6F74C01 | {"CAST_NAMESPACE":"urn:x-cast:tv.pluto","CAST_MESSAGE":{"command":"initializePlayback","channel":"","episode":"","time":0}} |
| Netflix                       | 3          | 1        | None     |
| YouTubeTV                     | 5          | 1        | None     |

> A current list of `app_id`'s can be found at http://hometest.buddytv.netdna-cdn.com/appservice/vizio_apps_prod.json and `NAME_SPACE` and `MESSAGE` values needed can be found in http://hometest.buddytv.netdna-cdn.com/appservice/app_availability_prod.json

## Misc

#### Status Responses
| `RESULT`                      |
| ----------------------------- |
| success                       |
| failure                       |
| uri\_not_found                |
| aborted                       |
| busy                          |
| blocked                       |
| requires_pairing              |
| requires\_system_pin          |
| requires\_new\_system_pin     |
| net\_wifi\_needs\_valid_ssid  |
| net\_wifi\_already_connected  |
| net\_wifi\_missing_password   |
| net\_wifi\_not_existed        |
| net\_wifi\_missing_password   |
| net\_wifi\_needs\_valid_ssid  |
| net\_wifi\_auth_rejected      |
| net\_wifi\_connect_timeout    |
| net\_wifi\_connect_aborted    |
| net\_wifi\_connection_error   |
| net\_ip\_manual\_config_error |
| net\_ip\_dhcp_failed          |
| net\_unknown_error            |

#### Types
| `ITEM`                      |
| --------------------------- |
| T\_UNKNOWN_V1               |
| T\_NO\_TYPE_V1              |
| T\_HIDDEN\_NETWORK_V1       |
| T\_DST\_LIST_V1             |
| T\_MENU_V1                  |
| T\_MENU\_X_V1               |
| T\_LIST_V1                  |
| T\_LIST\_X_V1               |
| T\_VALUE\_ABS_V1            |
| T\_VALUE_V1                 |
| T\_STRING_V1                |
| T\_PIN_V1                   |
| T\_IP\_ADDRESS_V1           |
| T\_MAC\_ADDRESS_V1          |
| T\_MATRIX_V1                |
| T\_HEADER_V1                |
| T\_ROW_V1                   |
| T\_DEVICE_V1                |
| T\_ACTION_V1                |
| T\_APS_V1                   |
| T\_AP_V1                    |
| T\_PASSWORD_V1              |
| T\_SOFTAP\_CONFIG_V1        |
| T\_LIST\_PAIRED\_DEVICES_V1 |
| T\_TEST\_CONNECTION_V1      |
| T\_IETF\_2822\_STRING_V1    |
| T\_DATE_V1                  |
| T\_LIST\_CEC\_DEVICE_V1     |
| T\_MANUAL\_IP\_CONFIG_V1    |
| T\_VIZIO\_DEVICE\_INFO_V1   |
| T\_UPDATE\_INFO_V1          |
| T\_ARRAY_V1                 |
| T\_EMAIL_V1                 |
| T\_LIST\_VALUES_V1          |
| T\_CEC\_DEVICE_V1           |

#### Additional Remote Codes
*ed: I can't see where these are used, but included anyway for completion.*

##### Note
Add a `MODIFIER` pair to `/key_command` request if used.

| Event Name   | Codeset | Code | Modifier |
| ------------ | :-----: | :--: | -------- |
| 0            | 0       | 48   |          |
| 1            | 0       | 49   |          |
| &            | 0       | 38   |          |
| *            | 0       | 42   |          |
| Backspace    | 0       | 8    |          |
| Bel          | 0       | 7    |          |
| Cancel       | 0       | 14   |          |
| ,            | 0       | 44   |          |
| $            | 0       | 36   |          |
| Esc          | 0       | 27   |          |
| !            | 0       | 33   |          |
| \            | 0       | 47   |          |
| -            | 0       | 45   |          |
| (            | 0       | 40   |          |
| Linefeed     | 0       | 10   |          |
| %            | 0       | 37   |          |
| .            | 0       | 46   |          |
| +            | 0       | 43   |          |
| #            | 0       | 35   |          |
| "            | 0       | 34   |          |
| Return       | 0       | 13   |          |
| )            | 0       | 41   |          |
| '            | 0       | 39   |          |
| Space        | 0       | 52   |          |
| Tab          | 0       | 9    |          |
| Audio MTS    | 5       | 5    | TYPE     |
| MTS Cycle    | 5       | 6    | TYPE     |
| Picture Mode | 6       | 0    |          |
| Wide Mode    | 6       | 1    |          |
| Wide Cycle   | 6       | 2    |          |

#### Additional Remote Code Sets

| Name         | Codeset |
| ------------ | :-----: |
| ASCII        | 0       |
| Key Modifier | 1       |
| Transport    | 2       |
| D-Pad        | 3       |
| Nav          | 4       |
| Audio        | 5       |
| Video        | 6       |
| Input        | 7       |
| CH           | 8       |
| Color        | 9       |
| Launch       | 10      |
| Power        | 11      |
| 3D           | 12      |
| CC           | 13      |
| Factor       | 14      |

