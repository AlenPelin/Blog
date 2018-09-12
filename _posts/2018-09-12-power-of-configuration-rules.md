---
title: Power of configuration rules
layout: post
---

## TL;DR;

Scroll down to [Environment Rules](#environment-rules) section that takes you through
`env:define/:require` and `whatever:/whatever:require` configuration out of
the box extensions.  

## Intro

Sitecore ecosystem is known for flexibility and numerous features 'out of the
box', but as we know everything comes with the price. Specifically for
`Sitecore Experience Commerce 9.0.2` the price includes approximate `350`
configuration files that combined consume enormous `3 MB` of `xml` text with
comments.

How much this pile of configuration will grow during real app development?
It is hard to say as it all depends on numerous factors such as scale, variety
of environments, development practices and tools used in process. And just like
in real life, when something is growing it's better to keep control of it. So
in this post I will try to describe fairly new successful way of organizing
configuration matter using configuration rules engine.

## Evolution of Roles

During whole lifetime of `Sitecore 6` developers had little issues with configs
due to size of them and limited set of roles: 

* `Content Delivery aka CD`
* `Standalone aka Content Management aka CM`  
  *Note while there could be several CD instances in the solution,
  statistically there always was only single CM (also known as Standalone in
  this case). Later these two roles became very different.* 

The most typical developer's challenge was to understand the main concept
itself that application may need more than one working instance at a time, and
how to `switch master to web`. 

As Sitecore product was getting maturer the demand on splitting system into smaller chunks was growing even faster. Until `Sitecore 7.5` there were two main roles that and two optional:

* `Dedicated Publishing`
* `Dedicated Email Dispatch` (for optional ECM/EXM module)

While `Sitecore 7.5` didn't get any major popularity, it was compensated by
`Sitecore 8.0` that flipped Sitecore world and made developers struggle with
enormous, overdetailed and often inconsistent documentation of converting
`Standalone` Sitecore instance type (which was preconfigured in distribution package with no other options) into one of following roles:

* `Content Management aka CM`
* `Processing`
* `Reporting`
* `Content Delivery aka CD`
* ... and various mixtures of first 3 of them:
  * `CM with Reporting`
  * `CM with Processing`
  * `CM with Processing and with Reporting aka Standalone`
* ... and various mixtures with:
  * `Dedicated Publishing`
  * `Dedicated Indexing` (which is worth separate article) 
  * `Dedicated Email Dispatch` (for optional ECM/EXM module)

## Sitecore-Configuration-Roles

Thanks to small initiative group with people from Sitecore Ukraine and Sitecore
Australia, `Sitecore 8.2.3` was released with unblocked configuration engine
that allowed to implement prototype
[Sitecore-Configuration-Roles](//github.com/alenpelin/sitecore-configuration-roles)
that extended default XML namespaces of Sitecore configuration with new one that
allowed annotating independent configuration elements with instructions when to
enable or disable them.

```xml
<configuration xmlns:role="http://www.sitecore.net/xmlconfig/role">
  <sitecore>
    <someElement someAttribute="someValue"
                 role:require="ContentManagement && !Processing">
      <!-- 
        the element itself and all children are processed only 
        when role:require="expr" expression is true 
      -->
```

There were a couple of key aspects that brought the project to success:

1.  Embracing familiar syntax of commonly used XML namespaces
2.  Providing annotated version of default Sitecore configuration files
3.  Support of a few partners who fearlessly used it in production
4.  Support of community for reporting issues and submitting fixes

## Configuration Rules

Followed by success of 
[Sitecore-Configuration-Roles](//github.com/alenpelin/sitecore-configuration-roles)
the initiative group acquired the prototype implementation and improved it, making
it even more helpful with generalized extensible configuration rules engine which is explained in 
[official documentation](//doc.sitecore.net/sitecore_experience_platform/developing/developing_with_sitecore/customizing_server_configuration/use_a_rulebased_configuration). Out of the
box Sitecore offers several important features:

* `role` in the very similar manner with configuration files annotated by default
* `search` provider which makes switching from Solr to Azure search in 5 keystrokes
* `exmEnable` lets switching EXM systems off when they are not required 
* `env` for environment pattern that lets replace MSBuild config transforms,
  or at least limit their usage only to the `web.config` file.

I'd like to talk more about `env:define` which terribly lacks documentation
(unlike two more other more obvious options), but first let's talk about mechanics
of the rule based config reader which is defined as follows:

```xml
<configuration>
  <configSections>
    <section 
      name="sitecore" 
      type="Sitecore.Configuration.RuleBasedConfigReader, Sitecore.Kernel" />
 ```

The logic of the system is straightforward:

1.  During website startup stage, create a matrix of defined categories of rules:  
      Check ASP.NET configuration for app settings in the given format:
      ```xml
      <configuration>
        <appSettings>
          <add name="banana:defined" value="african, american, australian"/>
          <add name="bird:defined" value="cockatoo|penguin"/>
          ...
      ```
      and form the in-memory dictionary of the following type:
      ```json
      {
        "banana": [ "african",  "american", "australian" ],
        "bird":   [ "cockatoo", "penguin",               ],
      }
      ```
2.  Extract `<sitecore>` node from ASP.NET configuration and merge config files
    from `App_Config` sub-folders according to `App_Config/Layout.config` file. 

    By default, the order looks like that:
    
    * `App_Config\Sitecore` (non-alphabetically, according to `Layouts.config`)
    * `App_Config\Modules` (alphabetically)
    * `App_Config\Include` (alphabetically)
    * `App_Config\Environment` (alphabetically)
  
3.  When recursively processing XML elements of particular `*.config` file, 
    check attributes for presence of `required` with XML namespace that follows the `http://www.sitecore.net/xmlconfig/KEYWORD/` pattern. If any of the rules
    fails, entire XML element with children is skipped. 

    Rule evaluation logic is simple: all defined tokens are replaced with `true`
    and the rest words - with `false`, and then expression is evaluated. 

    In accordance with banana-bird sample above, only <C> element will survive:
    
    ```xml
    <configuration 
      xmlns:banana="http://www.sitecore.net/xmlconfig/banana/"
      xmlns:horse="http://www.sitecore.net/xmlconfig/bird/">

      <sitecore>
        <A banana:require="(african or australian) and !american"/>
        <B banana:require="european" horse:require="cockatoo"/>
        <C banana:require="african" horse:require="cockatoo"/>
    ```

    **Important!** Even though it is a good practice to have namespace prefix match last word in actual schema URL, **technically it is not enforced**. 
    
    So in this sample `horse:` works with `bird:define` because of the appropriate
    namespace declaration: `xmlns:horse="http://www.sitecore.net/xmlconfig/bird/"`

## Environment Rules

As it is shown in previous [Configuration Rules](#configuration-rules) section,
Sitecore supports any `banana:define` and `banana:require` configuration rules
a developer finds useful. Out of box, there are a few of them used and only one
is not documented anywhere, even in the `web.config` itself.

In fact, there is a place in Sitecore files where it is used out of box and it
is the `App_Config/Environment/Sitecore.PipelineProfiling.config` file where
`env:require="Profiling"` rule is used. Quite interesting that 
[the official documentation](https://doc.sitecore.net/sitecore_experience_platform/developing/developing_with_sitecore/customizing_server_configuration/add_a_custom_rule_to_your_configuration)
mentions similar  `localenv:` token which can be a documentation issue and
it was supposed to use `env:` instead.

The sweetest part of the story is that you can mostly replace MSBuild transforms
usage with the `env:require` annotations in Sitecore patch files and only use
transforms with `web.config` to inject environment name:

```xml
<configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">
  <appSettings>
    <add key="env:define" value="Local, Win10, Development, Debug"
         xdt:Transform="Insert"/>
```

```xml
<configuration xmlns:xdt="http://schemas.microsoft.com/XML-Document-Transform">
  <appSettings>
    <add key="env:define" value="Azure, UAT, Debug"
         xdt:Transform="Insert"/>
```

What types of configuration parts can be controlled with `env:require`? Please
share you thoughts via DM on
[LinkedIn](//www.linkedin.com/in/alenpelin) or 
[Twitter](//twitter.com/alenpelin). 
The list will be extended with your samples, current set is:

* Site hostnames
* Unicorn mode and sourceFolder
* MailServer localhost or SendGrid
* [FridayCore](//github.com/alenpelin/FridayCore) features

There are things that probably shouldn't be controlled with configuration rules:

* `Credentials, Keys and Tokens` - keep them in safe encrypted place 
  (like Azure KeyVault), or at least in MSBuild transforms so accessing UAT won't
  compromise PROD keys.

In fact, there is not much difference between 

Google advises that this technique is not new and people were blogging about same thing about a year ago:

* [Sitecore 9 : Custom Config Roles](https://jammykam.wordpress.com/2017/11/01/sitecore-9-custom-config-roles/)
* [A Practical Approach to Structuring Sitecore 9 Config Files](https://www.degdigital.com/insights/sitecore-9-config-files/)
* [Getting rid of transformations, allow the same build to run in any environment](https://blogg.knowit.se/experience-se/getting-rid-of-transformations)

Apart from that, author managed to chat with several other people who confirmed
Sitecore solutions that successfully went live with this concept.
