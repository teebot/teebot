---
title: ASP.NET MVC, Injecting & Mocking an IRepository
date: 2010-03-03 16:30 UTC
tags: .NET, ASP.NET MVC, C#
---

If you are already familiar with ASP.NET MVC, you have probably seen the [repository pattern](http://martinfowler.com/eaaCatalog/repository.html) in a few examples (such as [NerdDinner](http://nerddinnerbook.s3.amazonaws.com/Intro.htm)), maybe you even use it in your own app.  

One of the most frequent way to hot-plug a Repository class into a controller is to use the Strategy Pattern. But while an application evolves, you might want to centrally manage your wiring within a provider, that's what Dependency Injectors are for.  
However lots of Dependency Injectors - or IoC containers at large - such as Unity or Castle Windsor contain a plethora of features and often rely heavily on XML configuration, making them overkill for smaller projects.  
[Ninject](http://www.ninject.org/) makes it relatively effortless to set up a DI. It just reached version 2.0.  

In this post we will see how to:  

*   Quickly inject a Repository in a controller with Ninject
*   Mock the IRepository interface in your tests with the Ninject.Moq plugin

**Get the bits**  

Here's where to get these dependencies:  
[Moq](http://code.google.com/p/moq/downloads/list)  
[Ninject2 (with Ninject.Web.Mvc)](http://ninject.org/download)  
[Ninject.Moq plugin](http://github.com/enkari/ninject.moq) (download and build)  

**Creating a Simple Repository**  

Let's start a quick MVC app with a Repository.  
The Winter Olympics provide us with a simple Model composed of the following types:  

[![](https://4.bp.blogspot.com/_sMzr7iNm7bs/S46ThH_XOmI/AAAAAAAAAX0/NvwZQcbP4GM/s400/ClassDiagram.png)](http://4.bp.blogspot.com/_sMzr7iNm7bs/S46ThH_XOmI/AAAAAAAAAX0/NvwZQcbP4GM/s1600-h/ClassDiagram.png)  

Our Repository class looks like this:  

```csharp
public class Repository
{
    public IEnumerable<Athlete> Athletes
    {
        get
        {
            return FakeDb.Athletes;
        }
    }

    public Athlete GetAthleteByName(string name)
    {
        var ath = from a in FakeDb.Athletes
                  where a.LastName == name
                  select a;
        return ath.Single();
    }

    public IEnumerable<Athlete> GetAthletesByDiscipline(string discipline)
    {
        var ath = from a in FakeDb.Athletes
                  where a.Discipline.Name == discipline
                  select a;
        return ath;
    }

    public IEnumerable<Athlete> GetAthletesByCountry(string country)
    {
        var ath = from a in FakeDb.Athletes
                  where a.Country.Name == country
                  orderby a.Discipline.Name
                  select a;
        return ath;
    }

}


```

Of course the first thing we'll do is to extract an IRepository interface from this class. We can at this point create a few Action Methods in an AthletesController to send some data to the view.  

```csharp

public interface IRepository
{
    IEnumerable<Athlete> Athletes { get; }
    Athlete GetAthleteByName(string name);
    IEnumerable<Athlete> GetAthletesByCountry(string country);
    IEnumerable<Athlete> GetAthletesByDiscipline(string discipline);
}

```

**Injecting an IRepository into a Custom Controller**  

We'll now create a custom Controller by extending the MVC Controller class. Our extended type will contain our injected IRepository and all our controllers will inherit from it. Notice the [Inject] attribute:  

```csharp

public class OlympicsController : Controller
{
    [Inject]
    public IRepository Repository;
}
```

We only need two things to set up Ninject and bind the Repository implementation as an IRepository in our controller instance:  

-First, a **Ninject Module class** where the binding occurs:  

```csharp


namespace WinterOlympics
{
  public class WebModule : NinjectModule
  {
      public override void Load()
      {
          

          //Repository injection
          Bind<iRepository>()
              .To<repository>();
          
      }
  }
}
```

-Second, we need to change our <span style="font-style: italic;">Global.asax</span> so our MvcApplication inherits from **NinjectHttpApplication** instead. That's also where the override <span style="font-style: italic;">CreateKernel()</span> references our binding module. Finally, we'll move the <span style="font-style: italic;">Application_Start</span> content under the <span style="font-style: italic;">OnApplicationStarted</span> override and call <span style="font-style: italic;">RegisterAllControllersIn()</span> with the current assembly as parameter.  

```csharp
namespace WinterOlympics
{
    
    public class MvcApplication : NinjectHttpApplication
    {
        public static void RegisterRoutes(RouteCollection routes)
        {
            routes.IgnoreRoute("{resource}.axd/{*pathInfo}");

            routes.MapRoute(
              "Default",
              "{controller}/{action}/{id}",
              new { controller = "Home", action = "Index", id = "" }
            );

        }

        protected override void OnApplicationStarted()
        {
            AreaRegistration.RegisterAllAreas();

            RegisterRoutes(RouteTable.Routes);
            RegisterAllControllersIn(Assembly.GetExecutingAssembly());
        }

        protected override IKernel CreateKernel()
        {
            return new StandardKernel(new WebModule());
        }
    }
}

```

Ninject is now set up. At any time during development you can switch your IRepository implementation by changing the bindings of the NinjectModule.  

**Mocking an IRepository with Ninject**  

Because our Repository implementation uses some sort of database, we don't want our Unit Tests to depend on it. It is generally recommended to test persistence layers in separate integration tests.  

For our Unit Tests, we set up a fake IRepository implementation with Moq.  
The Ninject.Moq plugin integrates our mock in our controller by using a <span style="font-style: italic;">MockingKernel</span> (as opposed to the StandardKernel previously seen in our <span style="font-style: italic;">Global.asax</span>).  

Let's create a test context in which we set up the mocked injection.  
Notice how the kernel injects a mock in the controller and how we're then able to retrieve the mock to set it up.  

```csharp
public class AthletesControllerContext
{
    protected readonly MockingKernel kernel;
    protected readonly AthletesController controller;
    protected readonly Mock<irepository> repositoryMock;

    public AthletesControllerContext()
    {
        kernel = new MockingKernel();
        controller = new AthletesController();
        //inject a mock of IRepository in the controller 
        kernel.Inject(controller);
        repositoryMock = Mock.Get(controller.Repository);

        //setup mocks
        Athlete mockAthlete = new Athlete()
        {
            BirthDate = new DateTime(1980, 5, 26),
            Discipline = new Discipline()
            {
                Name = "curling",
                IsPlayedInTeams = true
            },
            Country = new Country() { Name = "Luxemburg" },
            FirstName = "tee",
            LastName = "bot"
        };
        repositoryMock.Setup(m => m.Athletes).Returns(new[] { mockAthlete });
        repositoryMock.Setup(m => m.GetAthletesByCountry("Luxemburg")).Returns(new[] { mockAthlete });
        repositoryMock.Setup(m => m.GetAthletesByDiscipline("curling")).Returns(new[] { mockAthlete });
        repositoryMock.Setup(m => m.GetAthleteByName("teebot")).Returns(mockAthlete);
    }
}
``` 
[Thanks to [Miguel Madero](http://miguelmadero.blogspot.com/) for [helping me](http://groups.google.com/group/ninject/browse_thread/thread/00b002be89027fe9/77a5de8fbf599438?#77a5de8fbf599438) figure out this last part]  

Our test class inherits from this context and our test methods mimic the behavior of the application, goal achieved.  

```csharp

[TestClass]
public class AthletesControllerTest : AthletesControllerContext
{
    [TestMethod]
    public void Index_Returns_NotNull_ViewResult_With_Data()
    {
        //Act
        ViewResult res = controller.Index() as ViewResult;
        
        //Assert
        Assert.IsNotNull(res.ViewData);
    }

    [TestMethod]
    public void Detail_Returns_ViewResult_With_Data()
    {
        //Act
        ViewResult res = controller.Detail("teebot") as ViewResult;

        //Assert
        Assert.IsNotNull(res.ViewData);
        Assert.IsInstanceOfType(res.ViewData.Model, typeof(Athlete));
    }

    [TestMethod]
    public void Detail_Returns_NotFound_View_For_NonExisting()
    {
        //Act
        ViewResult res = controller.Detail("nonexisting") as ViewResult;

        //Assert
        Assert.AreEqual("NotFound", res.ViewName);
    }
}

``` 

[Download Sample Code](http://dev.teebot.be/blog/downloads/WinterOlympics.zip)

[Requires [Visual Studio 2010 RC](http://msdn.microsoft.com/en-us/vstudio/dd582936.aspx) / [ASP.NET MVC2 RC2](http://aspnet.codeplex.com/releases/view/39978)]  

**Wrapping Up**  

In this post we saw how to inject our Repository implementation as an IRepository member of the controller.  

Finally we switched our implementation for a mocked Repository in our tests.  

We clearly saw a case for injection as our application is now loosely coupled and our MVC is testable. We could switch from a database to an XML file as persistence layer without affecting the client or the tests.  

TeeBots-NET-Blog-ASPNET-MVC-Mocking-the-Repository-with-NinjectMoq)  
[CodeProject](http://www.teebot.be/2010/03/aspnet-mvc-and-ninject-repository.html)