---
title: Anatomy of a Phishing Email
date: 2020-10-27T19:18:42-05:00
draft: false
categories:
  - infosec
tags:
  - infosec
  - reversing
  - malware
toc: true
---

## Intro

We all get spam. Most of it is devoured by our mail providers' spam catcher, and we never see it. Every now and then, one slips through the cracks. In this case, I received an email earlier this week with a subject of "Re: Notification your test results COVID-19 [ note-7893 ]". Classic, making me think it's in reply to one of my email...that I sent about their test results?

> Dear Patients,
>
> Your SARS-CoV-2 Test results ready to be take off.
> 
> My office will call you to make an appointment so we can address this. If you have questions before your appointment, please call my nurse, Carolyn, at 425-277-1311.
> 
> Thank you and talk to you soon,
> 
> Attachment: Test Results
> Password/PIN for your Documents is: butts
> 
> Sincerely,
> Your Current, Retired and Future Doctors and Nurses

## Analysis

### Static

"Test Results" was a link to a URL shortener that pointed to a Firebase store. It downloaded a password-protected rar file. The password was as specified in the email. Clever way to avoid your payload getting scanned.

Inside the archive were 2 files:

```
╰─$ unrar l Test_Results.rar

UNRAR 6.00 beta 1 freeware      Copyright (c) 1993-2020 Alexander Roshal

Enter password (will not be echoed) for Test_Results.rar:

Archive: Test_Results.rar
Details: RAR 5, encrypted headers

 Attributes      Size     Date    Time   Name
----------- ---------  ---------- -----  ----
*   ..A....      1828  2020-10-26 18:34  Patient_Information.xml_;.lnk
*   ..A....    269312  2020-10-26 17:59  reportingresults.pdf
----------- ---------  ---------- -----  ----
               271140                    2
```

The shortcut file tries to be clever for those who hide file extensions and call itself an XML file. It's actually a shortcut to cmd to execute the pdf file. That's right, the PDF file is actually a PE32 .NET app -- at least according to our friend `file`. And indeed, the first 2 hex bytes are good 'ol `0x4d5a`.

Passing it into our good friend ILSpy, it's predictably obfuscated. Interestingly enough, it's a WinForms app. Embedded in `resx` files are some small `.bmp` files that don't seem to show anything interesting. Incidentally, [GCHQ Cyberchef](https://gchq.github.io/Cyberchef) is super handy for fiving deep and figuring out _what_ something is. (Yes, that it's base64 is obvious; less obvious is the `.bmp` signature). There's a variety of obfuscated strings in the `resx` files, some of which appear to be deobfuscated in code.

### Dynamic

With these sorts of obfuscated things, if you can't sort it out with something like `de4dot`, you'll often have an easier time debugging the damn thing. So, we set out into `dnSpy` to see what we find.

It ends up reading its own manifests (suprise), and constructs a `ResourceReader`. Turns out, there's a lot of extraneous crap (read: red herrrings) in there, so it constructs an `Enumerator` to go through them (this all via string obfuscation and calling methods via reflection).

Surprise, surprise, it grabs a second-stage payload. Interestingly enough, this second stage, while being a .NET application once again, also contains some assembly to do the nasty. Ends up being the Razy ransomware.

Possibly more to come?
