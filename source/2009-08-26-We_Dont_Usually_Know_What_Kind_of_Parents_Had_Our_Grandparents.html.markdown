---
title: We Don't Usually Know What Kind of Parents Had Our Grandparents
date: 2009-08-26 12:30 UTC
tags:
---

(or How to Retrieve a Type of Parent Control Using an Extension Method)  

Here's a situation where I had to retrieve the next Parent Control of a certain type. In this particular case, my label is included in an UpdatePanel to update only when my Label is assigned a new value.  

I could have used the Parent attributes of my controls and go up a few levels until I have the right one:  

```csharp
UpdatePanel updatePanel = label.Parent.Parent;
updatePanel.Update();
```

```html
The ASP:  
<asp:updatepanel id="UpdatePanelMessage" runat="server" updatemode="Conditional">
  <contenttemplate>
    
  </contenttemplate>
</asp:updatepanel>
```

**Do you see what's wrong?**

And what if the page is modified and the control is included in a new div or in a new panel? You see it coming, the control retrieved won't be an UpdatePanel because we don't usually know what kind of Parents had our Grandparents...  
So as soon as the asp page structure changes, chances are that casting errors start to appear.  

We need a way to look up that control's ancestry and return the first parent of a certain type. We can achieve that by using the following [extension method](http://msdn.microsoft.com/en-us/library/bb383977.aspx):  


```csharp
public static Control GetParentOfType(this Control childControl, Type parentType) { 
  Control parent = childControl.Parent; 
  while(parent.GetType() != parentType) { 
    parent = parent.Parent; 
  } 
  if (parent.GetType() == parentType) 
    return parent;  

  throw new Exception("No control of expected type was found"); 
}
```  

Now wherever we need to retrieve the first occurence of any control's parent of a certain type we just call the method `GetParentOfType` this way:  

```
UpdatePanel updatePanel = (UpdatePanel)label.GetParentOfType (typeof(UpdatePanel));
updatePanel.Update();  
```
  

[CodeProject](http://www.teebot.be/2009/08/extension-method-to-get-controls-parent.html)