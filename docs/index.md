---
layout: default
title: Home
---

# Documentation

Welcome to the documentation for the Focals Focals Developer API developer portal.
This guide will help you get started developing your own unique abilities for Focals smartglasses.

# What is the Focals Developer API?

The abilities frameowkr is the first SDK to bring Intelligent and data-rich experiences to life within our interaction model without writing code on Focals directly.

* Language Agnostic

* Deploy and host abilities ANYWHERE.

* Send notifications and other packets to devices

* Specify flexible web-powered actions that can be taken in response to notifications

# Using the developer portal

If you already have a focals account linked to a device, you should use that.  It will make testing your ability easier.
[Go here and login or create an account] (developer.bynorth.com)
Once you login (or after you validate your e-mail, if its a new account) you will need to accept the developer portal End-User License Agreement (EULA).

# Ability Credentials

Each ability has three keys used for authentication between the ability server and the Focals Developer API.
* API Key
* API Secret
* Shared Secret

Your ability will need to store all three of these values.
The first two (API Key and API Secret) are used together to authenticate when [publishing a packet to a user](#publish-to-user).

Shared Secret on the other hand, is used when receiving requests initiated by a user, such as:
* Enable ability for user
* Webhook from device
* Get User Settings

# How to validate shared secret

Whenever an HTTP request is made to the following routes, the request will include a `sharedSecret` query parameter.
* Webhook URL
* Settings URL

Your ability should compare this value to the shared secret provided by the developer portal to ensure that the request originated from The Focals Developer API.

Activation requests sent to the Activation URL request will include a HMAC signature.  Your ability should compute its own HMAC and compare to the provided value.
The signature is computed as follows:
1. Build a string of this form `${enableSigningVersion}:${timestamp}:${state}`
* enableSigningVersion = 'v0'
* timestamp = Time of the request in UNIX time (seconds since Jan 1, 1970 https://en.wikipedia.org/wiki/Unix_time)
* state = opaque session token (provided as a query parameter to the Activation URL)
2. Calculate the SHA-256 HMAC (https://en.wikipedia.org/wiki/HMAC) of this string using `shared secret` as the secret key.
3. Convert this HMAC to a hex digest string.
4. Compare calculated HMAC to provided signature

Here is a sample of signature validation in javascript:

```js
        const abilitiesSigningVersion = 'v0';
        const sharedSecret = config.get('ability_shared_secret');
        const { timestamp, state, signature } = req.query;
        const computedHmac = crypto
            .createHmac('sha256', sharedSecret)
            .update(`${abilitiesSigningVersion}:${timestamp}:${state}`)
            .digest('hex');

        if (`${abilitiesSigningVersion}:${computedHmac}` !== signature) {
            return res.redirect('https://cloud.bynorth.com/v1/api/integration/enable?error=InvalidSignature');
        }

```
# Packets

Packets are the primary way that abilities are empowered.  They have two main components:
1) Data sent to user to display on Focals
2) Actions that can be taken in response to this packet.

At the time of this document there is only one supported packet type, `actionable_text`, which is shown on the device as a notification.
When a user responds to the notification by choosing an action, the device will POST to your ability's Webhook URL and pass along the body provided by the packet.

Packets are described with json in accordance with this schema:

```js
{
        $schema: 'http://json-schema.org/draft-07/schema#',
        $id: 'http://cloud.bynorth.com/abilities/packet.v0.json',
        definitions: {
            action: {
                description: 'One action that the user can initiate on this packet',
                type: 'object',
                properties: {
                    type: {
                        description: 'Type of action.  Must be from list of supported Focals Developer API actions',
                        type: 'string',
                        enum: ['system:reply', 'system:webhook', 'system:group'],
                    },
                    actionId: {
                        description: 'Id of the action.  Should be unique within the actions for this packet.' +
                            'Can be displayed to the user if there is no title provided',
                        type: 'string'
                    },
                    title: {
                        description: 'Human readable title of the action.  Displayed to user',
                        type: 'string'
                    },
                    icon: {
                        description: 'Icon to display with the action',
                        type: 'object',
                        properties: {
                            type: {
                                description: 'Type of icon represented by the `value` property',
                                type: 'string',
                                enum: ['URL'],
                            },
                            value: {
                                description: 'Value of the icon',
                                type: 'string'
                            }
                        },
                        required: ['type', 'value']
                    },
                    subActions: {
                        description: 'Only used then type is `system:group`.  Array of actions associated with the group',
                        type: 'array',
                        items: {
                            allOf: [
                                {
                                    type: 'object',
                                    $ref: '#/definitions/action',
                                },
                                {
                                    properties: {
                                        type: {
                                            description: 'Type of sub-action.  Must be from list of supported Focals Developer API actions.  Cannot be a group',
                                            type: 'string',
                                            enum: ['system:reply', 'system:webhook'],
                                        }
                                    }
                                }
                            ]
                        }
                    }
                },
                required: [
                    'type',
                    'actionId'
                ],
                if: {
                    properties: { type: { enum: ['system:group'] } }
                },
                then: {
                    required: ['subActions'],
                    anyOf: [
                        { required: ['title'] },
                        { required: ['icon'] }
                    ]
                },
                else: {
                    if: {
                        properties: { type: { enum: ['system:webhook'] } }
                    },
                    then: {
                        not: { required: ['subActions'] },
                        anyOf: [
                            { required: ['title'] },
                            { required: ['icon'] }
                        ],
                    },
                    else: {
                        not: { required: ['subActions'] }
                    }
                },
                additionalProperties: false
            },
        },
        title: 'Packet',
        description: 'A packet sent from Focals Developer API to a user',
        type: 'object',
        properties: {
            templateId: {
                description: 'Id of the template',
                type: 'string',
                enum: ['actionable_text'],
            },
            icon: {
                type: 'object',
                description: 'The icon associated with the packet',
                properties: {
                    type: {
                        description: 'The type of url provided in the `value` property.',
                        type: 'string',
                        enum: ['URL'],
                    },
                    value: {
                        description: 'Icon value',
                        type: 'string'
                    }
                },
                required: ['type', 'value']
            },
            packetId: {
                description: 'Identifies the packet.  Will be passed back to integration with action responses, allowing ' +
                    'integrations to connect responses to a larger scope or context',
                type: 'string'
            },
            title: {
                description: 'Title of the actionable text.  Will be displayed to user',
                type: 'string'
            },
            body: {
                description: 'Contents of the actionable text.  Will be displayed to user',
                type: 'string'
            },
            timestamp: {
                description: 'Time when the packet was sent',
                type: 'string',
            },
            actions: {
                description: 'List of possible templated actions the user can initiate on this packet',
                type: 'array',
                items: {
                    $ref: '#/definitions/action',
                }
            },
        },
        required: ['templateId', 'icon', 'packetId', 'title', 'body', 'timestamp', 'actions']
    }
```

<a id="publish-to-user"></a>
# How to send a packet to a user

1. Construct the JSON packet
1. Encrypt the packet using the North abilities client library (https://github.com/focalsbynorth/tree/master/abilities-library/library) 
1. POST the encrypted packet to the publish to user route, found here https://cloud.bynorth.com/v1/api/integration/secure/publish-to-user

# Supported action types

At this time there are three types of actions supported by the Focals Developer API.
1. system:reply
1. system:webhook
1. system:group

# Handling actions
Your ability can have as many webhook actions are you like.  They are identified by strings (`actionId`).

Whenever a user takes an action Focals will send an HTTP POST request to your ability's Webhook URL.
The POST body will always have a paramter named `type` that identifies the action. Your ability should check the `type` parameter and take appropriate action.

For the most part you can name actions however you like, and there is no limit to how many action types an ability can possess.  The only restriction is that there are a few reserved action names, listed in the next section.

## Actions icons
There is a list of icons that can be used for actions.  
The syntax required in the action packet is:
`static:/system/icon/<icon-name>`

[For a full list of icons see the icons page here.](icons.md)

# Reserved integration action ids

The following action types are reserved by the system and have special meanings.
* integration:disable
* integration:validate
* integration:settings
* system:reply

Unlike other actions, these ones are not triggered in response to an action sent in a packet, but have special triggers.

## <a id="validate-action"></a>integration:validate

This action is triggered as the final step of activating an ability.  It is mandatory to implement this action, without it
a user cannot activate the ability.

It will convert a transient [state token](#state-token) to a user id.

It sends two parameters in the POST body
* state
* userId

In response, your ability should perform the following steps:
* Lookup the record from your unvalidated user storage and ensure the state is valid
* Make a record of the persistent userId in local storage
* Respond with HTTP 200 - OK

After this action is triggered, it is permitted to send packets to the provided userId.  Make sure to keep the userId in persistent storage.


## integration:disable

This action is mandatory and must be implemented by all abilities.  It is triggered when a user requests to disable the ability via the abilities page on the Focals Mobile App.
It will send one parameter in the POST body:
* userId

In response, your ability should perform the following steps:
* disable this user and cease sending packets to them.
* delete sensitive data related to the user, such as login tokens with 3rd party services, or PII
* respond with 200 - OK

Note that once a user is disabled, the Focals Developer API will reject packets published to that user with HTTP 403 - Forbidden.  Note that excessive messages sent to disabled users may result in your ability getting rate-limited or delisted from the abilities page.  Abilities should always honour a user's disable requests.

## integration:settings

When a user makes changes to their settings on the ability page, a JSON body will be sent to the webhook url with actionId set to `integration:settings`.  Handling this action is only necessary when your ability has a `Settings URL`.

The POST body will contain two parameters:
* userId
* settings

The ability is responsible for storing these settings in persistence storage.  Settings are a JSON object described by this schema:

```js
{
        $schema: 'http://json-schema.org/draft-07/schema#',
        $id: 'http://cloud.bynorth.com/abilities/settings.v0.json',
        title: 'Settings',
        description: 'Settings relating to an ability',
        type: 'object',
        properties: {
            version: {
                description: 'The version of the schema',
                type: 'string'
            },
            sections: {
                description: 'List of sections of settings',
                type: 'array',
                minItems: 1,
                maxItems: 3,
                items: {
                    description: 'One section of settings',
                    type: 'object',
                    properties: {
                        title: {
                            description: 'Title of this section of the settings',
                            type: 'string',
                            maxLength: 36
                        },
                        settings: {
                            description: 'The settings for this section',
                            type: 'array',
                            minItems: 1,
                            maxItems: 6,
                            items: {
                                description: 'A setting item',
                                type: 'object',
                                properties: {
                                    title: {
                                        description: 'Title of the setting',
                                        type: 'string',
                                        maxLength: 36
                                    },
                                    type: {
                                        description: 'The type of the setting',
                                        type: 'string',
                                        enum: ['toggle', 'dropdown']
                                    },
                                    options: {
                                        description: 'The values of the dropdown setting',
                                        type: 'array',
                                        minItems: 1,
                                        maxItems: 12,
                                        items: {
                                            description: 'Dropdown list item',
                                            type: 'string',
                                            minLength: 1,
                                            maxLength: 24
                                        }
                                    },
                                    description: {
                                        description: 'The description of this setting',
                                        type: 'string',
                                        maxLength: 36
                                    },
                                    allowEmpty: {
                                        description: 'Is a blank/empty value allowed',
                                        type: 'boolean'
                                    }
                                },
                                if: {
                                    properties: {
                                        type: {
                                            enum: ['toggle']
                                        }
                                    }
                                },
                                then: {
                                    properties: {
                                        value: {
                                            description: 'The current value for this setting',
                                            type: 'boolean'
                                        }
                                    },
                                    required: ['title', 'type', 'value']
                                },
                                else: {
                                    value: {
                                        description: 'The current value for this setting',
                                        type: 'string',
                                        maxLength: 24
                                    },
                                    required: ['title', 'type', 'value', 'options', 'allowEmpty']
                                }
                            }
                        }
                    },
                    required: ['title', 'settings']
                }
            }
        },
        required: ['sections', 'version']
    }
```

## system:reply

System reply is a special variant of webhook actions, where the device will automatically populate the POST body's `responseText` field.
At present, this information can come from three sources:
* voice to text
* smart reply
* smart emojis

The response will be sent to the webhook URL with actionId set to `reply`.

# Action Groups

Related actions can be grouped together into a group, using the special `system:group` meta-action.  A `system:group` action has a special
array property called `subActions` which contain a list of actions to associate with the group.  In the user interface on the device, all sub actions
will be shown together in a collection.

Groups have no impact on behaviour of the actions, and are used purely for front-end organizational purposes.

See the packet [schema](https://cloud.dev.bynorth.com/v1/abilities/schema/packet) for more details.

# Activation URL

When a user requests to enable an ability on the abilities in Focals Mobile App, an HTTP GET request will be sent to the ability's Activation URL.  The request will include one query parameter:
* state

<a id="state-token"></a>State is an opaque token that identifies the user for a short period of time, just long enough to sign the user up for the ability.

When your ability receives a request on this route it should perform whatever application logic is necessary to setup the ability for the user.  This can include redirecting the client to a third party login page when an integration with an external service is required.

In other cases it is as simple as storing the `state` token in a cache of unvalidated users.

When the above steps are complete, the ability server should redirect the client to this url using an HTTP 302 response code: https://cloud.bynorth.com/v1/api/integration/enable?state=${state}.

This will trigger the Focals Developer API to complete the enable flow by sending a validate action containing the `userId`.  See [integration:validate](#validate-action) for details

If something goes wrong with the enable flow, redirect them to the same URL but include an error query parameter with a short string describing the problem.

e.g.
https://cloud.bynorth.com/v1/api/integration/enable?&error=Failed%20to%20enable%20ability

This error message will be displayed to the user.

# Settings URL

The settings URL is used for retrieving settings for a user.
It will be called along with two query parameters:
* userId
* sharedSecret

When handling a request to this route it is recommended to check that the shared secret matches the shared secret provided by The Focals Developer API before responding.

This route should return a JSON object corresponding to the settings for this user.  [Schema](https://cloud.dev.bynorth.com/v1/abilities/schema/settings).
# Abilities page

The abilities page is shown on the Focals Mobile app.  It contains a list of all abilities available to the user, along with their current status (enabled/disabled).  When users activate an ability on this page a GET request is made to the ability's Activation URL.  The ability is responsible for displaying any pages necessary to setup the ability, including OAUTH login redirects, if applicable.

# Publishing and collaborators

An ability can be in one of three stages:
* dev
* review
* public

A new ability always starts in `dev` and it is only available to the developer who created it.

If more than one person needs to work on it, add a collaborator by entering their e-mail address into the collaborators section on your abilities' dashboard.  The ability will then be made visible on the collaborator's ability page.

When an ability is fully tested and ready to be release to all Focals users, click the publish button on the ability's dashboard.


# Hosting a web application

Ability services can be hosted anywhere as long as they are publicly available.
Examples:
* [Heroku](https://devcenter.heroku.com/start)
* [ngrok](https://ngrok.com/)

# Focals Developer API Encryption

The Focals Developer API uses a flexible end to end encryption scheme.
When sending messages from your ability to Focals, sensitive fields are encrypted using the device's public key, and can only be decrypted by the device.
When Focals sends messages to your ability, it is encrypted using your ability's public key, and can only be decrypted using your ability's private key.

In both directions of data flow, the same encryption scheme is used.
The encryption scheme details can be found [here](encryption.md)

## Encryption:  Packet fields

When encrypting an ability packet, the following fields may be encrypted:
* packetId
* icon.value
* title
* body

The actions array must be plain, however the following action fields may be encrypted:
* actionId
* title

## Encryption: Public Key

Before your ability can receive encrypted messages from Focals, it is necessary to generate a key pair.

A key pair can be generated with the following openssl commands on OSX:

```
brew install openssl
openssl genrsa 4096 > ability.privatekey.pem
openssl rsa -in ability.privatekey.pem -pubout -out ability.publickey.pem
```

The above commands generate two files, containing the private and public keys respectively.  Keep your private key in a safe place.  The North Ability Node.js Client Library can be configured with your private key in order to easily decrypt messages.
