---
title: Setting AD Properties using the new AccountManagement API
date: 2009-11-16 16:30 UTC
tags: .NET, ActiveDirectory, C#
---

So I'm currently programming a lightweight utility to manage LDAP objects and I came to use the new [AccountManagement API](http://msdn.microsoft.com/en-us/library/system.directoryservices.accountmanagement.aspx) introduced in .NET 3.5 because it's way easier and more object-oriented than [DirectoryServices](http://msdn.microsoft.com/en-us/library/system.directoryservices.aspx).  

This new namespace, as I see it, is just provided as a friendlier way to query an LDAP server and is actually built on top of the plain DirectoryServices bits. And it shows... For example you have to dispose all objects performing a search (FindIdentity) or as Gary Caldwell [puts it](http://msdn.microsoft.com/en-us/library/bb345628.aspx):  

> This call has an unmanaged memory leak because the underlying implemenation uses DirectorySearcher and SearchResultsCollection but does not call dispose on the SearchResultsCollection as the document describes.

Also, because AccountManagement is not as close to the metal as DirectoryServices you can only control whatever Microsoft chose to expose in the API.  

However a fantastic feature of AccountManagement is the ability to fall back on a DirectoryServices object whenever you need to because the underlying object of any Principal is exposed through GetUnderlyingObject().  

```csharp
DirectoryEntry entry = (DirectoryEntry)principal.GetUnderlyingObject();
```

With that much power in our hands we can now use this to set some properties unexposed by AccountManagement. Here's a generic method I made to set LDAP properties on any Principal (users, groups, computers):  



```csharp
public void SetCustomProperty<T>(string key, string value, T principal)
 where T : Principal {
 if (principal.GetUnderlyingObjectType() == typeof(DirectoryEntry)) {
   DirectoryEntry entry = (DirectoryEntry)principal.GetUnderlyingObject();
   entry.Properties[key].Value = value;
  
   try
   {
      entry.CommitChanges();
   }
   catch (Exception e)
   {

      string message = 
      String.Format(@"Could not set custom property {0} to value {1} for object {2}", 
      key, value, principal.Name);

      throw new Exception(message, e);
   }
 }
}
 ```
  

Use it like this:  

```csharp
SetCustomProperty<userprincipal>("streetAddress", "21 jumpstreet", userPrincipal);
```
  

Being user-friendly while keeping control? I'm starting to like that little guy.  

**UPDATE: As an anonymous commenter pointed out, there's a way to actually map an AD attribute to a property of a class inheriting UserPrincipal. The technique consists of decorating the property with the attribute <span style="font-style: italic;">DirectoryProperty</span> and the field name as parameter (as seen [here](http://msdn.microsoft.com/en-us/magazine/cc135979.aspx#S10)).**  

**NOTE: code indentation is messy in the examples. I need to fix something in my snippet tool.**  


[CodeProject](http://www.teebot.be/2009/12/ad-setting-custom-properties-using-new.html)