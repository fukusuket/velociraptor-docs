---
title: Windows.Applications.Outlook.PST.Header
hidden: true
tags: [Client Artifact]
---

This artifact fetch emails and header details such as SPF, DMARC and DKIM from outlook PST file.


<pre><code class="language-yaml">
name: Windows.Applications.Outlook.PST.Header
author: "Sikha Puthanveedu @SikhaMohan"
description: |
  This artifact fetch emails and header details such as SPF, DMARC and DKIM from outlook PST file.
parameters:
  - name: outlookPSTfile
    type: .pst
    description: Full path to the outlook .pst file (For example - D:/MyPST/MyOutlookDataFile.pst)

sources:
  - precondition:
      SELECT OS FROM info() WHERE OS = 'windows'
    query: |
        
            LET PSTInfo = SELECT Sender as Sender,
                 Receiver as Receiver,
                 Subject as Subject,
                 Message as Message,
                 DateandTime as DeliveryTime,
                 Attachments as AttachmentNames,
                 Body as Body
            from parse_pst(filename=outlookPSTfile)
          
            SELECT  
               Sender,
               Receiver,
               parse_string_with_regex(regex='spf=(\\w+)', string=Body).g1 AS SPF,
               parse_string_with_regex(regex='dkim=(\\w+)', string=Body).g1 AS DKIM,
               parse_string_with_regex(regex='dmarc=(\\w+)', string=Body).g1 AS DMARC,
               parse_string_with_regex(regex='Return-Path: &lt;(.*?)&gt;', string=Body).g1 AS ReturnPath,
               Subject,
               DeliveryTime,
               AttachmentNames,
               Message,
               parse_string_with_regex(regex='internet_message_id:"&lt;(.*?)&gt;"', string=Body).g1 AS msgId,
               parse_string_with_regex(regex='Content-Type:(.*?);', string=Body).g1 AS ContentType
            FROM PSTInfo 
</code></pre>

