# ASPNETCore-Fundamentas

- _Layout.cshtml is the principal page, here we can see the first tag helper

```C#
<a class="nav-link text-dark" asp-area="" asp-page="/Privacy">Privacy</a>
```

- asp-page tag helper will do is set the href or my archor tag, to point to a Razor Page that I have in my project. We are going to add a new link to the VideoGames/List view and then create the folder VideoGames within Pages folder and create inside an Empy Razor Page List.cshtml

- The system created two files, List.cshtml that is the Razor Page with the directive @page that tells ASP.NET Core that this is a Razor Page and has another directvie @model ListModel, this directive says that the model for this page is type ListModel (an instance of the ListModel class is the object that this page will use to display information.

- The second file is List.cshtml.cs and contains the class ListModel, the methods here are going to be very much like controller actions and the properties are going to be very much like the models that i would pass into a view.

- How we can interact between Razor Page and the Razor Page's PageModel? we have two ways, the firs is very simple in the model we are going go create a property and intialized in the OnGet method.

```C#
public class ListModel : PageModel
{
  public string Message { get; set; }
  
  public void OnGet()
  {
      Message = "Hello VideoGames!!";
  }
}
```

- And in the view only have to call the property by the Model

```HTML
<p>@ModelMessage.</p>
```

- The other way is by the application configuration, one of the sources of configuration for this application is appsettings.json. Anything I place into this file is something I can access at runtime.

```JSON
"Message":  "Hello from appsettings!"
```

- And now in the PageModel constructor take a parameter of type IConfiguration, add a private readonly property and then I have access to configuration throughout the rest of the PateModel.

```C#
private readonly IConfiguration _configuration;

public ListModel(IConfiguration configuration)
{
    _configuration = configuration;
}

public void OnGet()
{
    Message = _configuration["Message"];
}
```

- Now we are going to create our first Entity, for do it instead create a new folder for that, we are going to create a new project of class library type. Within create a VideoGame class and an enum for the companys.

```C#
public class VideoGame
{
    #region Public Properties

    public Company Company { get; set; }

    public int Id { get; set; }

    public string Name { get; set; }

    #endregion Public Properties
}
```

```C#
public enum Company
{
    Microsoft,
    Nintendo,
    PlayStation
}
```

- We need something that we can use for data access that will store and retrive VideoGame objects. I want to implemented something easy to swap with a real database when I finist my tests in development. For this pourpose, we're going to keep our data access classes separate from our entities and from the UI or the web application part. Create a new Class Library Project called JcProject.Data and create an interface called IVideoGameData.  
  For use the VideoGame class we have to add the reference to the JcProject.Core and we can see this in the JcProject.Data.csproj
  
  ```HTML
  <ItemGroup>
    <ProjectReference Include="..\JcProject.Core\JcProject.Core.csproj" />
  </ItemGroup>
```

- Create de interface and InMemory class for store the data

```C#
public interface IVideoGameData
{
    IEnumerable<VideoGame> GetAll();
}

public class InMemoryVideoGameData : IVideoGameData
{
    readonly List<VideoGame> videogames;
    public InMemoryVideoGameData()
    {
        videogames = new List<VideoGame>()
        {
            new VideoGame { Id = 1, Name ="Gears of war", Company = Company.Microsoft},
            new VideoGame { Id = 2, Name ="God of war", Company = Company.PlayStation},
            new VideoGame { Id = 3, Name ="Mario Sunshine", Company = Company.Nintendo},
        };
    }
    public IEnumerable<VideoGame> GetAll()
    {
        return videogames;
    }
}
```

- ASP.NET Core follows the dependency inversion principle and this allow to tell the framework that whenever a component like a Razor Page needs something that implements IVideoGameData, it should provide that component with InMemoryVideoGameData specifically.  And we can do this in the startup.cs file within ConfigureServices Method.

```C#
services.AddSingleton<IVideoGameData, InMemoryVideoGameData>();
```

- In the PageModel add the interface in the constructor, add a property to store the VideoGames data and use de OnGet method to simulate a http get request for get all the data.

```C#
private readonly IVideoGameData _videoGameData;

public ListModel(IConfiguration configuration, IVideoGameData videoGameData)
{
    _configuration = configuration;
    _videoGameData = videoGameData;
}

public IEnumerable<VideoGame> VideoGames { get; set; }

public void OnGet()
{
    VideoGames = _videoGameData.GetAll();
}
```

- In the view create a table for show the data

```HTML
<table class="table">
    <thead>
        <tr>
            <th>ID</th>
            <th>Name</th>
            <th>Company</th>
        </tr>
    </thead>
    <tbody>
        @foreach(var videogame in Model.VideoGames)
        {
        <tr>
            <td>@videogame.Id</td>
            <td>@videogame.Name</td>
            <td>@videogame.Company</td>
        </tr>
        }
    </tbody>
</table>
```

- We want to implement a search form to find a VideoGame by the name

```HTML
<form method="get">
    <div class="form-group">
        <div class="input-group">
            <input type="search" class="form-control" value="" name="searchTerm" />
            <span class="input-group-btn">
                <button class="btn btn-default">
                  <i class="glyphicons glyphicons-home"></i>
                </button>
            </span>
        </div>
    </div>
</form>
```

- A good practice is change te GetAll method by GetVideoGameByName passing a string parameter with the name.  In any case, if recived a empty string or null we return all the videogames.

```C#
public interface IVideoGameData
{
    IEnumerable<VideoGame> GetVideoGameByName(string name);
}
```

```C#
public IEnumerable<VideoGame> GetVideoGameByName(string name)
{
    return String.IsNullOrEmpty(name) ? videogames : videogames.Where(v => v.Name.Contains(name));
}
```

- In in the OnGet method we defined a parameter with the same name that the input field, ASP.NET Core when invoke that method is go out through the requests and try to find something that is a serchTerm, it will look in the query string, headers, etc.

```C#
public void OnGet(string searchTerm)
{
    VideoGames = _videoGameData.GetVideoGameByName(searchTerm);
}
```

- Now we want searchTerm to be both an input and an output property so that we can use its value also in the input value, for that we can use BindProperty --> this attribute tells the ASP.NET Core Framework when you're instantiating this class and you're getting ready to execute a method on this class to process an HTTP request, this particular property should receive information from the request.
By default ASP.NET Core is only going to bind input properties during an HTTP POST operation, but there is a flag I an use to control that behavior SupportsGet = true

```C#
[BindProperty(SupportsGet = true)]
public string SearchTerm { get; set; }

public void OnGet()
{
    VideoGames = _videoGameData.GetVideoGameByName(SearchTerm);
}
```

- In the view we can use now a tag helper for the binding asp-for --> is saying this input is for the following property that you'll find on the model

```HTML
<input type="search" class="form-control" asp-for="SearchTerm" />
```

- Detail page, we have to add a button for each videoGame to go to the detail page, in the List.cshtml view we are going to add a new td and use the tag helpers for tell to ASP.NET Core which is the page and which is the paremeter with the value

```HTML
<td>
    <a class="btn btn-lg"
       asp-page="./Detail" asp-route-videoGameId="@videogame.Id">
        <i class="glyphicon glyphico-zoom-in"></i>
    </a>
</td>
```

- Create the detail page

```C#
public VideoGame VideoGame { get; set; }

public void OnGet(int videoGameId)
{
    VideoGame = new VideoGame();
    VideoGame.Id = videoGameId;
}
```

- In the view

```HTML
@page
@model JcProject.Pages.VideoGames.DetailModel
@{
    ViewData["Title"] = "Detail";
}

<h2>Name: @Model.VideoGame.Name</h2>

<div>
    Id: @Model.VideoGame.Id
</div>
<div>
    Company: @Model.VideoGame.Company
</div>

<a asp-page="./List" class="btn btn-default">All VideoGames</a>
```

- If we want to pass the route VideoGames/Detail/1 instead /VideoGames/Detail?videoGameId=1 we can do it in the @page directive

```HTML
@page "{videoGameId:int}"
```

- Now we have to add the method to the interface

```C#
VideoGame GetById(int id);
```

- And implemented the method

```C#
public VideoGame GetById(int id)
{
    return videogames.SingleOrDefault(v => v.Id == id);
}
```

- We want to prevent and handling errors, for do it we have to return an IActionResult in the OnGet method. If doesn't error we can return a Page(), Page return a PageResult that implement IActionResult and tells ASP.NET Core that render the page or take the Detail.cshtml and render that

```C#
public IActionResult OnGet(int videoGameId)
{
    VideoGame = _videoGameData.GetById(videoGameId);
    if (VideoGame == null)
    {
        return RedirectToPage("./Error");
    }
    return Page();
}
```


