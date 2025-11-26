---
title: 'XSS to RCE via Upload File'
date: 2025-03-16
permalink: /posts/2025/03/firstBug-post/
tags:
  - cool posts
  - category1
  - category2
---

# Stored XSS and Remote Code Execution (RCE) via File Upload

During an analysis of the web application that we’ll call *example.com*, I identified two critical vulnerabilities: a **Stored XSS** and a **Remote Code Execution (RCE)**.

The goal of this article is to describe the process of identifying, demonstrating, and reporting the two vulnerabilities I discovered, highlighting the security issues commonly found in web applications within institutional contexts.

## Exploring the Application

Upon accessing [example.com](example.com), after creating an account and logging in, we are redirected to the homepage.

What immediately caught my attention was the **"My Data"** section, where a user can enter personal data, links to their social profiles, and even upload files.  
In particular, the section that intrigued me the most was this one:

![My Data](https://i.ibb.co/ZRzZJP5f/foto1.webp)

Here, the user can upload files that will later be displayed by clicking their corresponding blue buttons.

This feature made me think right away about the possibility of bypassing file extension checks. So I began testing by uploading files with extensions different from the allowed ones. After several attempts, I submitted this payload:

![Payload XSS](https://i.ibb.co/GvDg9xL8/xsspayload.webp)

Once the uploaded file was viewed, this was the result:

![XSS Triggered](/research/firstBug/xss.png)

**Boom! Stored XSS executed!**

## Attempting RCE

I didn’t stop there. I wanted to test whether it was possible to execute commands on the machine via a PHP file.  
So I attempted to upload a `.php` file containing the following payload:

![Payload RCE](/research/firstBug/rcepayload.png)

Once the file was displayed, I noticed something strange:

![File with .txt](/research/firstBug/rce.png)

The system automatically appended a `.txt` to the `.php` extension, preventing the execution of the code.

After several tests on how to bypass the final `.txt`, it came to my mind that PHP files can also have other valid extensions (such as `.php2`, `.php3`, etc.).  
So I tried sending the same payload with the `.php2` extension:

![Payload PHP2](/research/firstBug/rce2payload.png)

Once the file was displayed, here was the result:

![RCE Triggered](/research/firstBug/rce2.png)

**And there it is: PHP code executed successfully!**

## Conclusion

Both vulnerabilities found (Stored XSS and RCE via file upload) were reported and resolved.  
With this post, I want to highlight how important it is to properly validate and sanitize user input, especially in contexts where sensitive data is handled.  
I strongly recommend that developers review upload policies, allowed extensions, and correctly implement server-side security mechanisms.

If you have questions, doubts, or want to discuss this analysis, feel free to contact me. 😎
