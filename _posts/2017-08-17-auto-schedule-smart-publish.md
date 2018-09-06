---
layout: post
title: AutoScheduleSmartPublish
---

# 1. Publishing.AutoScheduleSmartPublish

From the very old versions Sitecore had the `Publishing.AutoScheduleSmartPublish` setting that when enabled makes publish engine to trigger full site smart publish when specific item is changed - based on defined template. By default, this setting is on and configured this way:

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

This is a good idea to switch this setting off when you don't want any unwanted publish happen.