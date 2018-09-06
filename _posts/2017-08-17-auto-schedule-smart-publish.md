---
layout: post
title: Don't get surprised by automatic publish
---

It came to us from the dark times when Sitecore was young and developers were brave.
In fact, it was decided that once one of the *special* items is modified the engine must start a publishing process immediately.
There were only a few types of these items, and these types were defined as follows:

```xml
<!-- PUBLISHING -->
<publishing>
  <smartPublishTriggers>
    <trigger templateId="{CB3942DC-CBBA-4332-A7FB-C4E4204C538A}" note="proxy" />
    <trigger templateId="{AB86861A-6030-46C5-B394-E8F99E8B87DB}" note="template" />
    <trigger templateId="{455A3E98-A627-4B40-8035-E683A0331AC7}" note="template field" />
  </smartPublishTriggers>
</publishing>
```

Luckily, as many other great things in Sitecore, this one can be disabled via `Publishing.AutoScheduleSmartPublish` setting.
This might be a good idea to switch it off in any case.