# dunp - Decentralized Universal Networking Protocol

## Idea

For a general social platform to use decentralized web is a bit overwhelming for developers and might represent a barrier for many users if not done well. We want to simplify that, by building the **dunp protocol** that acts as a social networking data layer, completely agnostic to the social media or content types. We also remove user acquisition friction by providing a login mechanism that allows to sign in with a simple username and password pair, without any crypto knowledge. The user stays in control of their data all times, so more power to users and interoperability between apps is now a reality. **dunp protocol** - the future of social networking for web3.

Twitter CEO, Jack Dorsey, July 23rd 2021:
*"I think there is a lot of innovation above just currency to be had, especially as **we think about decentralizing social media more** and providing more economic incentive."*

We say: *Want to decentralize your social media content? Just dunp it here!* :)

## Technologies

- **IPFS**: every instance is a js-ipfs node
- **OrbitDB**: takes care of the data over IPFS, each instance with same identity has the same access to user data and replicates it
- **Ethereum HD wallets**: each OrbitDB node has an identity derived from an Ethereum account, that can be generated by our easy login system - based on a username and password or signed by external signers like Metamask
- **Any EVM based blockchain**: can take care of the business model, as all data is provably signed by the owner of that account, but out of scope of this layer
- **web3.storage**, **nft.storage**, **Piñata**: apps can subscribe to any of the integrated IPFS+Filecoin storage providers, and have their users data replicated
- **Filecoin**: Used indirectly by the pinning/replication services to incentivise permanent data storage

### Innovations

- Users can login using username and password, and an ethereum account will be generated for them without them even knowing what a key is. Savvy users can use Metamask, or other wallet/signers in the future, but not mandatory.
- Ethereum keys/seeds/mnemonics are not saved ever. They are generated and used to sign a Decentralized IDentity (DID), then that identity is stored temporarily in browser to sign further user data. The user can choose to export mnemonic / keys when he so desires to improve data security.
- All user data belongs to the user, no exeptions. Apps are views over user data and incentive mechanisms applied to that data.
- All data is content addressed and id resolved by the peers, so that we can have shorter links in apps frontends.
- User data is organized in portfolios, by content type, so same content can be available to various apps that handle that type of content.
- Apps can run their resolvers, ipfs pin nodes and replicators, in order to improve network data availability, but not mandatory and they can never change or remove users data.
- Apps can choose to have their user data replicated to services like web3.storage or Piñata, to keep users data backed up and improve availability.
- dunp protocol is opensource, and provides an SDK for apps to use. This aims to facilitate the use of web3 technologies for all developers.


## Architecture

### @dunp/identity

This package handles creation of Decentralized IDentities and keys to access the protocol data layer.

### @dunp/identity/easy

Easy login translates a username and password pair to a deterministic seed that is supplied to create an Ethereum HD wallet. That address is then used as external public key to create the Decentralized IDentity and sign it.

This guarantees that the same username and password pair will generate always the same keys and will have access to the same databases of that identity.

### @dunp/identity/web3

This identity provider does the same but with a web3 wallet, like Metamask. This should be used by crypto savvy people, instead of easy login.

### @dunp/identity/...

Other identity providers coming soon... like wallet-connect, for instance [TBD]


### @dunp/data

This package handles the social networking data layer.

- **settings**

    App global settings data, to hold any configuration that is to be shared with all app instances, but only updatable by the app identity.

    Initialized by: {
        appid: string
        identity: string
    }

    name | type | write
    ---|---|---
    dunp.${appid} | keyvalue | app identity

    **data**:
    ```
    {
        version: string
        ...
    }

- **identity**

    Decentralized IDentity (DID) as from @dunp/identity flavors. This identity is what authorizes a user to manipulate their profile, portfolios and all shared content. The identity is used to create the dunp data instance, which is a factory to all other layers.

- **profile**

    Each identity has an associated profile.
    Note: there's no username - usernames or slugs are app dependent, so each app is responsible to maintain their verified accounts lists and decide on attributing usernames or slugs that link directly to a dunp profile and/or portfolio(s). Also apps might consider using ENS or Unstoppable Domains for their human readable pointers, and dunp supports any blockchain domain system that is able to resolve names to ethereum accounts.

    name | type | write
    ---|---|---
    dunp.profile | docs | identity

    **data**:
    ```
    {
        appid: string (index) 'default' for default profile that app specific override if found

        version: string
        created: timestamp
        updated: timestamp
        publish: bool

        id: nanoid
        name: string
        avatar: cid
        bio: string
        location: string
        links: {
            [website]: string
        }
        birthday: timestamp

        tags: [string]

        metadata: {}
    }
    ```

- **portfolio**

    Each profile has many portfolios identified by their type. Examples of portfolios can be: post, blog, audio, video, photo, book, event, course, shop, etc.
    For each portfolio there are types and categories to organize content, defined by the apps.
    For example, audio portfolio can have types: music, sample, podcast, asmr, etc. and category defining main genre in case of music.
    A collections portfolio is defined by appending `:set` to the portfolio, for instance, audio collections go into portfolio audio:set, then collection types can be: list (post:set), album (photo:set), album (audio:set), release (audio:set), playlist (video:set), playlist (audio:set), etc.
    There are also tags that can be any hashtag.

    name | type | write
    ---|---|---
    dunp.${portfolio} | feed | identity

    **data**:
    ```
    {
        id: nanoid
        version: string
        created: timestamp
        updated: timestamp
        publish: bool

        content: cid
        thumbnail: cid
        title: string
        summary: string
        length: number (seconds/pages/...)
        
        type: string
        category: string
        tags: [string]

        metadata: {
          ${app id}: { ... }
        }
    }
    ```

- **content**

    Each entry in a portfolio can have a content property, which is a CID (content id or multihash) of a directory added to IPFS. Other properties of the portfolio entry can then point to files inside that content directory. Each content directory has to have a file in it's root named `manifest.json`, that will describe how to render that content, and link back to the portfolio entry:
    ```
    {
        id: ${content id}
        portfolio: ${portfolio address}
        version: string

        mime: mimetype
        index: cid
    }
    ```

- **stats**

    Each profile has their stats databases by portfolio type and content id.

    name | type | write
    ---|---|---
    dunp.${portfolio}.${content id}.stats | keyvalue | identity

    **data**:
    ```
    {
        attention: number (seconds)
        feedback: number (positive/negative?)
        report: {} ?
        ...
    }
    ```

- **favorites**:

    Each profile has a list of favorite content by portfolio type.

    name | type | write
    ---|---|---
    dunp.${portfolio}.favorites | feed | identity

    **data**:
    ```
    {
        version: string
        publish: bool

        portfolio: address
        content: nanoid
    }
    ```

- **discussion** (content comments)

    Each content can have a public discussion panel, where anyone can join in by appending their comments database to this list.

    name | type | write
    ---|---|---
    dunp.${portfolio}.${content id}.discussion | eventlog | public

    **data**:
    ```
    {
        comments: address
    }
    ```


- **comments**

    Each comments database of each profile that joined the discussion above, has the comments of that profile in that discussion.
    This database has a new type created by dunp, CommentRateStore, that rate limits comments from the same user.

    name | type | write
    ---|---|---
    dunp.${portfolio}.${content id}.comments | discussion | identity

    **data**:
    ```
    {
        id: nanoid
        version: string
        created: timestamp
        updated: timestamp

        replyto: nanoid
        owner: nanoid

        ...
    }
    ```

- **subscriptions**

    Each profile can subscribe to other portfolios by portfolio type, for instance, one can follow the music of someone, but not his posts.

    name | type | write
    ---|---|---
    dunp.${portfolio}.subs | feed | identity

    **data**:
    ```
    {
        version: string
        publish: bool

        portfolio: address
    }
    ```

- **notifications**

    create the best notification mechanism...

- **processing**

    app specific organized by profile and portfolio, so that only wanted content is replicated. matches cid from original content to cid of processed content entry

    global database?

- **statistics**

    RFU
    Public database for content statistics, where everyone can add plays or other stats for each content on the network.

- **feeds**

    tags, type, categories, user, subs, etc... app specific algo

    RFU
    Generates a content feed to be rendered by the social app.


### @dunp/resolver

Responsible for resolving all types of objects by id, name, etc. into a database or content address. Uses pubsub as the communication layer between peers.
Enables profile discovery and content search in a decentralized way.


### @dunp/replicator

Interface to the pinning services to be integrated in dunp, so that all data gets replicated automatically.

### @dunp/replicator/web3.storage

### @dunp/replicator/nft.storage

### @dunp/replicator/pinata

