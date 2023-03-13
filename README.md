# LocalSend Protocol v2

## Table of Contents

- [1. Defaults](#1-defaults)
- [2. Fingerprint](#2-fingerprint)
- [3. Discovery](#3-discovery)
    - [3.1 Multicast](#31-multicast-udp-default)
    - [3.2 HTTP](#32-http-legacy-mode)
- [4. File transfer](#4-file-transfer-http)
    - [4.1 Send request](#41-send-request-metadata-only)
    - [4.2 Send file](#42-send-file)
    - [4.3 Cancel](#43-cancel)
- [5. Additional API](#5-additional-api)
  - [5.1 Info](#51-info)

## 1. Defaults

LocalSend does not require a specific port or multicast address but instead provides a default configuration.

Everything can be configured in the app settings if the port / address is somehow unavailable.

The default multicast group is 224.0.0.0/24 because some Android devices reject any other multicast group.

**Multicast (UDP)**
- Port: 53317
- Address: 224.0.0.167

**HTTP (TCP)**
- Port: 53317

## 2. Fingerprint

The fingerprint is used to avoid self-discovery and to remember devices.

When encryption is on (HTTPS), then the fingerprint is the SHA-256 hash of the certificate.

When encryption is off (HTTP), then the fingerprint is a random generated string.

## 3. Discovery

### 3.1 Multicast UDP (Default)

**Announcement**

At the start of the app, the following message will be sent to the multicast group:

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile", // mobile | desktop | web
  "fingerprint": "random string",
  "announce": true
}
```

**Response**

Other LocalSend members will notice this message and will reply with their respective information.

First, an HTTP/TCP request is sent to the origin:

`POST /api/localsend/v2/register`

```json5
{
  "alias": "Secret Banana",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string" // ignored in HTTPS mode
}
```

As fallback, members can also respond with a Multicast/UDP message.

```json5
{
  "alias": "Secret Banana",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string",
  "announce": false
}
```

The `fingerprint` is only used to avoid self-discovering.

A response is only triggered when `announce` is `true`.

### 3.2 HTTP (Legacy Mode)

This method should be used when multicast was unsuccessful.

Devices are discovered by sending this request to all local IP addresses.

`POST /api/localsend/v2/register`

Request

```json5
{
  "alias": "Secret Banana",
  "deviceModel": "Windows",
  "deviceType": "desktop",
  "fingerprint": "random string" // ignored in HTTPS mode
}
```

Response

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung",
  "deviceType": "mobile",
  "fingerprint": "random string" // ignored in HTTPS mode
}
```

## 4. File transfer (HTTP)

### 4.1 Send Request (Metadata only)

Sends only the metadata to the receiver.

The receiver will decide if this request gets accepted, partially accepted or rejected.

`POST /api/localsend/v2/send-request`

Request

```json5
{
  "info": {
    "alias": "Nice Orange",
    "deviceModel": "Samsung", // nullable
    "deviceType": "mobile", // mobile | desktop | web
    "fingerprint": "random string" // ignored in HTTPS mode
  },
  "files": {
    "some file id": {
      "id": "some file id",
      "fileName": "my image.png",
      "size": 324242, // bytes
      "fileType": "image", // image | video | pdf | text | other
      "preview": "*preview data*" // nullable
    },
    "another file id": {
      "id": "another file id",
      "fileName": "another image.jpg",
      "size": 1234,
      "fileType": "image",
      "preview": "*preview data*"
    }
  }
}
```

Response

```json5
{
  "sessionId": "mySessionId",
  "files": {
    "someFileId": "someFileToken",
    "someOtherFileId": "someOtherFileToken"
  }
}
```

### 4.2 Send File

The file transfer.

Use the `sessionId`, the `fileId` and its file-specific `token` from `/send-request`.

This route can be called in parallel.

`POST /api/localsend/v2/send?sessionId=mySessionId&fileId=someFileId&token=someFileToken`

Request

```text
Binary data
```

Response

```text
No body
```

### 4.3 Cancel

This route will be called when the sender wants to cancel the session.

Use the `sessionId` from `/send-request`.

`POST /api/localsend/v2/cancel?sessionId=mySessionId`

Response

```text
No body
```

## 5. Additional API

### 5.1 Info

This was an old route previously used for discovery. This has been replaced with `/register` which is a two-way discovery.

Now this route should be only used for debugging purposes.

`GET /api/localsend/v2/info`

Response

```json5
{
  "alias": "Nice Orange",
  "deviceModel": "Samsung", // nullable
  "deviceType": "mobile" // mobile | desktop | web
}
```