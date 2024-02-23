---
title: "Hello Read You: Welcome to Google Reader API"
tags:
  - ReadYou
  - Android
date: 2024-02-21
---

|English|[中文](https://Ash7.io/blog/hello-read-you-welcome-to-google-reader-api-zh-cn/)|

Two weeks ago, we released version [0.9.12](https://github.com/Ashinch/ReadYou/releases/tag/0.9.12) of Read You, which includes integration with the Google Reader API, as well as improvements to the user interface. We'd like to take this opportunity to share some of the insights and experiences we had while integrating the Google Reader API into Read You. This will help users understand some of the internal mechanisms of Read You and hopefully assist in the development of future RSS readers.

## Multi-Tenancy

A brief description of the multi-tenancy concept for RSS readers: allowing readers to have multiple (even different types of) accounts simultaneously while maintaining a consistent user experience. This solves issues related to local data backup, isolation, cloud synchronization, and also caters to a broader range of RSS user base.

Different types of accounts:

- **Local**: All data is stored only on the current device, and this reader meets the basic RSS usage requirements.
- **Self-hosted**: RSS tools deployed and hosted by users, which can be integrated with multiple supported readers to achieve cloud synchronization. These tools are mostly open-source projects, undergo code audits from the community, and users have full control over their data and privacy without relying on third-party services. Representative self-hosted RSS tools include FreshRSS, Miniflux, Tiny Tiny RSS, etc.
- **Third-party service providers**: Mature commercial products that integrate various RSS features, usually require internet connectivity, and have data and privacy controlled and protected by service providers. Users benefit from their cloud computing technology to experience more mature features such as real-time push notifications of new articles or commercial advertisements. Representative RSS service providers include Inoreader, Feedly, Feedbin, etc.

The original intention of developing Read You was to give Android its own "Reeder", so many design concepts come from this "masterpiece" on iOS, and the design of the multi-tenancy architecture is no exception.

These are the expectations for Read You after analyzing Reeder's multi-account design:

- **Data layer**: Use the unique identifier field of each tenant as the vertical segmentation point. Any query and modification of user behavior patterns will be accompanied by their own tenant identifier from the bottom layer to isolate the data between tenants and prevent data overflow.
- **Logical layer**: In the application behavior mode, except for the application layer, all other logics need to consider the problems existing in the multi-tenancy scenario.
- **UI layer**: Design different characteristics according to tenant types, abstract common characteristics into public components, and differentiate characteristics into separate components, paying attention to providing targeted state management.

Read You started preparing for multi-account design since its inception. Currently, Read You has a good grasp of the commonalities of various account types in terms of UI/UX and has made some differentiation based on the API supported by different account types, such as Fever accounts that do not support write operations.

However, there are also some reconcilable issues in database modeling (due to previous haste). As for the current table structure, a better design should be: the IDs of Article, Feed, and Group should all be UUIDs, used only for local CRUD operations. In addition, a separate External ID field should be provided for idempotent control. For local account types, this value can be values such as ID or GUID in RSS 1.0, RSS 2.0, Atom.

This pit is particularly tricky when supporting integration with third-party services. After supporting the import/export function of article data, this function must be refactor as soon as possible, although it will inevitably involve database migration again.

## Google Reader from 16 Years Ago

Google has never publicly released the API documentation for Google Reader. The various documents circulating on the market today are the results of reverse engineering at that time.

An Article in Google Reader is one type of Item concept. When passing the ID of an Item to the server, to reduce the size of the message body, the ID is generally a long integer:

```
150177826473082
```

However, when the server returns the details of the Item to the client, the Item ID is generally in a specific format:

```
tag:google.com,2005:reader/item/000088960000047a
```

The first part `tag:google.com,2005:reader/item/` is a fixed value, and the second part `000088960000047a` is derived from `150177826473082` by converting it to a unsigned zero-padded 64 bit hexadecimal number format, corresponding to Kotlin code:

```kotlin
fun String.ofItemIdToHexId() = String.format("%016x", toLong())
```

In Google Reader, the Tag concept is a more general operation for filtering and organizing information, such as:

- Adding a Tag to an Article can mark it as starred or read.
- Adding a Tag to a Feed makes feeds with the same Tag form a Folder (aka Group or Category).

It is obvious that a Feed can be added with multiple Tags, and the relationship between Feed and Folder in Google Reader is many-to-many. If the reader integrated with it does not support multiple Folder relationships, some targeted changes need to be made, such as only taking the first Folder corresponding to the Feed.

In Google Reader, organizing and filtering information results in various Streams:

- **All Items**: `user/-/state/com.google/reading-list`
- **Starred Items**: `user/-/state/com.google/starred`
- **Read Items**: `user/-/state/com.google/read`
- **Folder, Tag, or Smart**: `user/-/label/...`
- **Feed**: `feed/...`
- **Broadcast** (third-party service providers focusing on articles generally return empty results): `user/-/state/com.google/broadcast`

When querying Streams via the Google Reader API, flexible set operations can be used to further narrow down the query range, reduce server pressure, and reduce message size. For example, to retrieve 100 unread Items added to the server after `2024-02-20 12:00:00`:

```
GET /stream/items/ids
?s=user/-/state/com.google/reading-list
&xt=user/-/state/com.google/read
&ot=1708401600
&n=100
```

## Various "Google Readers"

There are many third-party service providers that have implemented the Google Reader API in accordance with the Google model. FreshRSS, BazQux, etc., have relatively comprehensive support, while the implementations of other third-party service providers may differ:

- Inoreader: Now requires OAuth 2.0 authentication.
- Miniflux: When the authentication interface passes `output=json`, the Response is a JSON structure, which is understandable. However, currently, some providers implement this authentication interface and, for compatibility reasons, choose to ignore the `output` parameter passed to the authentication interface, just like Google Reader, so their Response is `text/plain`, and their field values are separated by newline characters.

Today, service providers that support the Google Reader API have only implemented some of its interfaces, which are generally commonly used and indispensable during the entire synchronization process. When integrating these interfaces, attention needs to be paid to the extent to which various service providers support these interfaces, and sometimes compromises need to be made in terms of interface selection for compatibility.

Taking `/mark-all-as-read` as an example, this interface allows marking up to 50k Items in the Stream as read at one time, but unfortunately not all implemented service providers support this interface. Therefore, only the most common and best-supported `/edit-tag` interface can be selected to mark Items as read in batches.

## Synchronization Strategy

Compared to other APIs, the Google Reader API offers more freedom, and there are multiple implementation options during the synchronization process.

The synchronization process mainly includes:

1. **Syncing Tags**: `/tag/list`
2. **Syncing Folders**: `/subscription/list`
3. **Syncing Feeds**: `/subscription/list`
4. **Syncing Articles and their Status**

The implementation methods of various Readers for the first three points are basically the same, but for the fourth point, there may be significant differences in implementation due to differences in complexity, conditions, and emphasis.

Some considerations when synchronizing articles and their status:

> How far back to sync articles?

Imagine if there are 100k articles on the server dating back three years, what kind of disaster scenario would it be to completely sync for the reader?

> Do we need all unread articles?

This seems unquestionable. We want to sync unread articles to let users read, but is it necessary to sync unread articles from three years ago?

> Do we need all starred articles?

Starred articles are crucial "assets" for users, and of course, we hope that the reader can sync all starred articles on the server for users to browse at any time. But it's another matter if there are 50k starred articles on the server.

Each reader has its own answers to these questions. This [Issue](https://github.com/FreshRSS/FreshRSS/issues/2566#issuecomment-541317776) discusses some readers' synchronization strategies, and developers are also looking for their ideal "best practices".

Read You has referenced some readers' "answers" to the Google Reader API, and after comprehensive domain modeling and database design, it adopted the same [synchronization strategy](https://github.com/Ashinch/ReadYou/pulls?q=is:pr+is:closed+greader) as Reeder due to its similarity to Reeder's situation.

Reeder's synchronization strategy: It syncs complete starred articles, complete unread articles, and some read articles within the past month, providing a good reading experience for users across multiple devices. Users can always get the content they want to sync.

At the initial sync cycle, Read You fetches all read articles from the past month to provide a better experience across multiple devices. However, in subsequent sync cycles (the `n`-th time), it only fetches articles that differ between the local device and the server, thus reducing unnecessary network requests.

Currently, most of the time spent on syncing in Read You is spent on replacing existing subscription database records. This can be solved by modifying the model in the future. Syncing relies mainly on memory to exchange data, which may not be very friendly for low-memory devices. This problem can be solved by using a database intermediate table for data exchange. However, other logic in Read You has already consumed too many connection pool resources. Increasing connection pool consumption would lead to a lagging experience even on high-memory devices. We also look forward to solving these problems through model refactoring in future iterations.

Detailed synchronization process of Read You:

1. Get Tags on the server (since Read You's Tag feature has not been implemented yet, this step is skipped for now).
2. Get the list of subscription sources and their associated Folders on the server (if there are missing data locally, use the data from the server to fill in the gaps, keeping it idempotent with the server).
3. Get the set of IDs of all unread items on the server.
4. Get the set of IDs of all starred items on the server.
5. Take the set difference between the set of IDs of unread items on the server and locally, then use the IDs in the difference set to further retrieve the corresponding article content (up to 10k items at a time).
6. Take the set difference between the set of IDs of starred items on the server and locally, then use the IDs in the difference set to further retrieve the corresponding article content (up to 10k items at a time).
7. Take the set difference between the set of IDs of read items within the past month on the server and locally, then use the IDs in the difference set to further retrieve the corresponding article content (up to 10k items at a time).

If there are a large number of unread articles, starred articles, or read articles within the past month on the server, it will definitely significantly increase the initial synchronization time. However, once synchronized, the data difference between local and server will become smaller, so the next synchronization will be much faster.

It can be seen that this synchronization strategy depends more on the Reader's "periodic sync" feature. By regularly synchronizing to keep the data difference between local and server small, each automatic and manual synchronization only requires minimal time.

## Parting Words

During the process of integrating the Google Reader API into Read You, we faced many challenges and made significant improvements. We look forward to the next technical sharing.

Best regards,
Ash 👋
