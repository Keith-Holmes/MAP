# Magic Attribute Protocol
### A system for mapping arbitrary hyper media to a global identifier, or anything.

## PROTOCOL
```
<OP_RETURN | <input>>
MAP
<SET | DELETE>
<key>
<value>
```

#### Use cases
- map a comment to a url
- map an action to a txhash (like, repost, or flag a comment)
- map a photo to a geolocation
- map a 'type' to some data (this data is a 'post')
- map ______ to a _______

#### PROTOCOL CHAINING

MAP is designed to be chained together with other OP_RETURN micro-protocols. The input stream flows from the left and can be piped like unix commands. We chain content from B protocol, and map it to some global identifier (txhash, url etc), and finally sign that using the Author Identity protocol:

    B | MAP | AUTHOR_IDENTITY

More on protocol piping:
https://github.com/unwriter/Bitcom/issues/2

# Examples
## SET: Comment on a URL
Here is the full piped OP_RETURN sequence for mapping B data to a global ID.

```
OP_RETURN B | MAP SET 'url' 'https://map.sv' | AUTHOR_IDENTITY_PROTOL
```

A more detailed view of the same transaction. Each line here is a new pushdata:

```
OP_RETURN
19HxigV4QyBv3tHpQVcUEQyq1pzZVdoAut (B)
"## Hello small world"
text/markdown
utf8
|
1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5 (MAP)
'url'
'https://map.sv'
|
15PciHG22SNLQJXMoSUaWVi7WSqc7hCfva (AUTHOR_IDENTITY)
1
ecdsa
<pubkey>
<signature>
 ```

## Setting Multiple Keys: User Profile
Keys and values can be repeated to set multiple attributes at once:
```
1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5 (MAP)
SET
'coolapp.profile.link'
'http://mywebsite.com'
'coolapp.profile.name'
'username123'
```

## DELETE: Remove Profile Data
To delete one of the keys->value mappings from the example above.
```
1PuQa7K62MiKCtssSLKy1kh56WWU7MtUR5 (MAP)
'DELETE'
'coolapp.profile.name'
```

# Concepts
## Keys are Namespaces

Since the keyspace is shared, you can either prefix your keys with a unique identifier, or operate in the global space, sharing that dataset and inheriting the emergent schema. Sharing the global naimspace can be useful when it is intended to be shared among many apps.

  Potential Namespaces for Global Identifiers

    url
    tx
    ethtx
    btctx
    topic
    upc
    infohash
    ifps
    isbn
    md5

Some global identifiers have more than one value...

  *Coordinates*

    coordinates.lat = 
    coordinates.lng =
    coordinates.alt =

  *Phone*

    phone.country_code = 1
    phone.url = 9549549544

  *Profile*

    profile.pubkey = "1HQ..."
    profile.name = "Satchmo"
    profile.text = "Hello small world!"
    profile.image = "b://98bcef1cc43ae..."
    profile.banner = "https://www..."
    

## Keys are Actions
If the above example is namespace as a noun, in this example we show a namespace can be used as a verb too. Actions usually dont need input data. Instead you act upon something that already exists. These begin new op_return chains instead of taking input from a previous protocol. This is useful if all you need is a single key to begin the chain, such as 'like'.

#### Comparison to Memo Protocol for Actions

Like something by txid:
```
Memo
0x6d04	txhash(32)

MAP
MAP 'like' 'true' tx <txhash>
```

Set your profile
```
Memo (multiple txs)
1. Set name
  0x6d01 <name>(217)

2. Set profile text
  0x6d05 <message>(217)

3. Set profile picture
  0x6d0a <url>(217)

MAP
MAP SET 'profile.name' 'Satchmo' 'profile.text' 'Some cool text' 'profile.picture' 'b://986...'
```

Follow / Unfollow users

```
Memo
Follow user	0x6d06	address(35)		
Unfollow user	0x6d07	address(35)		

MAP
MAP 'follow.user' <address>
MAP 'unfollow.user' <address>
```

Follow / unfollow topic
```
Memo
Topic follow	0x6d0d	<topic_name>(variable)		
Topic unfollow	0x6d0e	<topic_name>(variable)	

MAP
MAP 'follow.topic' <topic_name>
MAP 'unfollow.topic' <topic_name>

```
## Attach Content
#### Memo Commands with Content

Post
```
  Memo
  0x6d02  <message>(217)	

  MAP
  B <message> <content-type> <encoding> | MAP 'post' 'psot'
```

Reply to Tx
```
  Memo
  0x6d03  <txhash>(32)  message(184)	

  MAP
  B <message> <content-type> <encoding> | MAP 'reply' <txhash> | AUTHOR_IDENTITY
```

Repost
```
  Memo
  0x6d0b  <txhash>  <message>

  MAP
  B <message> <content-type> <encoding> | MAP 'repost' <txhash> 
```

Topic Post
```
  Memo
  0x6d0c  topic_name(variable)  message(214 - topic length)	

  MAP
  B <message> <content-type> <encoding> | 'topic' <topic_name>
```

## MAP is Powerful - More Use Cases

Comment on a URL "anonymously"
```
B <message> <content-type> <encoding> | MAP url http://google.com
```

Comment with an identity
```
B <message> <content-type> <encoding> | MAP url http://google.com | AUTHOR_IDENTITY
```

Attach a picture to a geolocation
```
B <image> <content-type> <encoding> | MAP 'coordinates.lat' <latitude> 'coordinates.lng' <latitude> 'coordinates.alt' <altitude>
```

Comment on a phone number
```
B <message> <content-type> <encoding> | MAP 'phone.country_code' <country_code> 'phone.number' <phone_number>
```

Comment on a UPC code
```
B <message> <content-type> <encoding> | MAP 'upc' <upc_code>
```

# SCRIPT SCHEMA
https://github.com/unwriter/Bitcom/issues/3

We can define the protocol name as well as the field names. v1 of script scema does not support variadic arguments... For now let's assume we have a hypothetical v2 schema that supports variadic arguments like this:

```
{
  "v": 2,
  "s": {
    "out.s1": "{{map}}
    "$repeat": {
      "out.si": "{{key}}",
      "out.si": "{{value}}"
    }
  }
}
```

Then register your schema on chain...

    `OP_RETURN MAP 'appname.schema' 'b://txid' | AUTHOR_IDENTITY`

## Querying NameSpaces
Thanks to Script Schema + A custom BitDB Planaria, we can create an api that allows you to query by protocol and field names. It can also pre-process identity signatures to make sure you only index transactions with valid signature identity, so your frontend doesnt need to do the validation work.

`protocol: { field: xxx }`

Query for all url posts from a particular user:
```
{
  v: 1,
  q: {
    find: {
      map: {                          // <- protocol name is auto-mapped by MAP planaria
        key: 'url'
      },
      author: {                       // <- protocol name
        pubkey: <pubkey>
      }
    }
  }
}
```

Find all likes mapped to a geolocation
```
{
  q: {
    find: {
      map: {
         $and: [{
          'coordinates.lat': '40.68433',
          'coordinates.lng': '-74.39967'
         },{
          'like': {'$or': ['true', 'false']}
         }]
      }
    }
  }
}
```

## A Complete Complex Example: Creating Polls

This is how memo protocol deals with polls. As you can see it takes a few transactions to create a poll:

| Command         | Prefix | ARG1                | ARG2              | ARG3              |
|-----------------|--------|---------------------|-------------------|-------------------|
| Create poll     | 0x6d10 | `<poll_type>`(1)    | `<option_count>`  | `<question>`(209) |
| Add Poll Option | 0x6d13 | `<poll_txhash>`(32) | `<option>` (184)  |                   |
| Poll vote       | 0x6414 | `<poll_txhash>`(32) | `<comment>` (184) |                   |


This is how we could do them with B, MAP, and AUTHOR in 1 transaction:

```
B
'best cheese?'                   <- poll (question)
'text/markdown'
'utf8'
|
MAP
'poll.type'
<'one'|'many'>
'poll.options'
'cheddar'
'poll.options'
'swiss'
'poll.options'
'xrp'
'poll.options'
'xmr'
'poll.options'
'gouda'
|
AUTHOR_IDENTITY
<version>
<algorithm = ecdsa|pgp|... >
<pubkey>
<signature>
```

Technically you could add the poll options as an array and process 

And this is how we can query for the relavent data:
```
{
  q: {
    find: {
      map: {
        key: 'poll'
      }
    }
  }
}
```

and you should get a response like this:
```
[{
  b: {
    content: 'best cheese?',
    content-type: 'text/markdown',
    encoding: 'utf8'
  },
  map:{
    'poll.type': 'one'
    'poll.options': ['cheddar'
      'cheddar',
      'swiss',
      'xrp',
      'xmr',
      'gouda'
    ]
  },
  author: [{
    <pubkey>
  }]
}, ...]
```

#### Vote on a poll
```
OP_RETURN
MAP
'poll.vote.tx'
<poll_txid>
'poll.vote.option'
<option_index>
```

And to find votes for a poll:
```
{
  q: {
    find: {
      map: {
        key: 'poll.vote.tx',
        value: <poll_txid>
      }
    }
  }
}
```

and you should get a response with many transactions, each looking like this:
```
{
  map: {
    poll: {
      vote.option: '1',
      vote.option: '2'
    }
  },
  author: {
    pubkey: <pubkey>,
    ...
  }
}
```