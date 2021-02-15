# ASPNETCore-Fundamentas

- _Layout.cshtml is the principal page, here we can see the first tag helper

```C#
<a class="nav-link text-dark" asp-area="" asp-page="/Privacy">Privacy</a>
```

- asp-page tag helper will do is set the href or my archor tag, to point to a Razor Page that I have in my project. We are going to add a new link to the VideoGames/List view and then create the folder VideoGames within Pages folder and create inside an Empy Razor Page List.cshtml

- The system created two files, List.cshtml that is the Razor Page with the directive @page that tells ASP.NET Core that this is a Razor Page and has another directive @model ListModel, this directive says that the model for this page is type ListModel (an instance of the ListModel class is the object that this page will use to display information).

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
        return RedirectToPage("../Error");
    }
    return Page();
}
```

- And modify the Error page

```HTML
@page
@model ErrorModel
@{
    ViewData["Title"] = "Error";
}

<h2 class="text-danger">Your Videogame was not found</h2>

<a asp-page="./VideoGames/List" class="btn btn-primary">See all restaurants</a>
```

- Edit page, copy and paste the td for ./Detail and change for ./Edit

```HTML
  <td>
      <a class="btn btn-lg"
         asp-page="./Edit" asp-route-videoGameId="@videogame.Id">
          <i class="glyphicon glyphico-edit"></i>
      </a>
  </td>
```

- The Edit.cshtml.cs is very similar like the Detail

```C#
 public class EditModel : PageModel
{
    private readonly IVideoGameData _videoGameData;

    public EditModel(IVideoGameData videogameData)
    {
        _videoGameData = videogameData;
    }

    public VideoGame VideoGame { get; set; }

    public IActionResult OnGet(int videoGameId)
    {
        VideoGame = _videoGameData.GetById(videoGameId);
        if (VideoGame == null)
        {
            return RedirectToPage("../Error");
        }
        return Page();
    }
}
```

- Remember the asp-for tag helper can do at least two jobs --> One is to set the name attribute of this input so that when the form is submitted, ASP.NET model binding can say this value is the value that should be placed into de VideoGame.Id property.  asp-for can also set the value of this imput.

- In the select - option, whe don't want to hardcode, then we can use the asp-items tag helper and has two properties, one property says, here's the text to display, and the other property says, here's is the value to submit when that particular entry is chosen.  We need the data in the CompanyType enum and we can use IHtmlHelper

```C#
private readonly IHtmlHelper _htmlHelper;

public EditModel(IVideoGameData videogameData, IHtmlHelper htmlHelper)
{
    _videoGameData = videogameData;
    _htmlHelper = htmlHelper; 
}

public IEnumerable<SelectListItem> Companies { get; set; }

 public IActionResult OnGet(int videoGameId)
{
    Companies = _htmlHelper.GetEnumSelectList<CompanyType>();
    VideoGame = _videoGameData.GetById(videoGameId);
    if (VideoGame == null)
    {
        return RedirectToPage("../Error");
    }
    return Page();
}
```

- The view would look like this

```HTML
@page "{videoGameId:int}"
@model JcProject.Pages.VideoGames.EditModel
@{
    ViewData["Title"] = "Edit";
}

<h2>Editing @Model.VideoGame.Name</h2>

<form method="post">

    <input type="hidden" value="" asp-for="VideoGame.Id" />
    <div class="form-group">
        <label asp-for="VideoGame.Name"></label>
        <input asp-for="VideoGame.Name" class="form-control" />
    </div>

    <div class="form-group">
        <label asp-for="VideoGame.Company"></label>
        <select asp-for="VideoGame.Company" asp-items="Model.Companies" class="form-control">
        </select>
    </div>

    <button type="submit" class="btn btn-primary">Save</button>
</form>
```

- Creating http post, we have to add new methods in the interface, the OnCommit method is thinking when we data source will be Entity Framework

```C#
VideoGame Update(VideoGame updatedVideogame);

int Commit();

public int Commit()
{
    return 0;
}

public VideoGame Update(VideoGame updatedVideogame)
{
    var videoGame = videogames.SingleOrDefault(v => v.Id == updatedVideogame.Id);
    if (videoGame != null)
    {
        videoGame.Name = updatedVideogame.Name;
        videoGame.Company = updatedVideogame.Company;
    }

    return videoGame;
}
```

- And now we need to create the OnPost method in the Edit.cshtml.cs, First we have to add the BindProperty to the VideoGame property, then when when the user click on the save button this videogame should be populated with information from the form.

```C#
[BindProperty]
public VideoGame VideoGame { get; set; }

public IActionResult OnPost()
{
  Companies = _htmlHelper.GetEnumSelectList<CompanyType>();
  _videoGameData.Update(VideoGame);
  _videoGameData.Commit();
  return Page();
}

```

- Adding validation, int the VideoGame class

```C#
[Required, StringLength(80)]
public string Name { get; set; }
```

- Change the OnPost method

```C#
public IActionResult OnPost()
{
    if (ModelState.IsValid)
    {
        _videoGameData.Update(VideoGame);
        _videoGameData.Commit();

    }
    Companies = _htmlHelper.GetEnumSelectList<CompanyType>();
    return Page();
}
```

- In the view add the asp-validation-for tag helper

```HTML
<span class="text-danger" asp-validation-for="VideoGame.Name"></span>
```

- In web development is a bad idea to leave the user on a page that is displaying the results of an HTTP POST operation. Because if refresh the page, send other HTTP POST, in our case we can add a redirect to detail page bellow commit line.

```C#
return RedirectToPage("./Detail", new { videoGameId = VideoGame.Id });
```

- Adding create option, we are going to use the edit page for edit and create new videogame. First we need to add a new button on the list view.

```HTML
<a asp-page="./Edit" class="btn btn-primary">Add New</a>
```

- In the Edit view we are going to change the int parameter to optional parameter

```HTML
@page "{videoGameId:int?}"
```

- In the OnGet method we change de parameter to optional, if has value populate de videogame with the data, if doesn't have value create an empty videogame

```C#
public IActionResult OnGet(int? videoGameId)
{
    Companies = _htmlHelper.GetEnumSelectList<CompanyType>();
    if (videoGameId.HasValue)
    {
        VideoGame = _videoGameData.GetById(videoGameId.Value);
    }
    else
    {
        VideoGame = new VideoGame();
    }

    if (VideoGame == null)
    {
        return RedirectToPage("../Error");
    }
    return Page();
}
```

- In the Interface add a new method

```C#
VideoGame Add(VideoGame newVideoGame);

public VideoGame Add(VideoGame newVideoGame)
{
    videogames.Add(newVideoGame);
    //This line is only for simulate a database save
    newVideoGame.Id = videogames.Max(v => v.Id + 1);
    return newVideoGame;
}
```

- Refactor the OnPost method

```C#
public IActionResult OnPost()
{
    if (!ModelState.IsValid)
    {
        Companies = _htmlHelper.GetEnumSelectList<CompanyType>();
        return Page();
    }

    if (VideoGame.Id > 0)
    {
        _videoGameData.Update(VideoGame);
    }
    else
    {
        _videoGameData.Add(VideoGame);
    }
    _videoGameData.Commit();
    return RedirectToPage("./Detail", new { videoGameId = VideoGame.Id });
}
```

- Adding confirm operation, we can pass a message like a temporaly string witn TempData, add this below commit in the OnPost method

```C#
TempData["Message"] = "VideoGame saved!";
```

- In the Detail.cshtml.cs add a new property

```C#
[TempData]
public string Message { get; set; }
```

- And in the Detal view add a conditional

```HTML
@if (Model.Message != null)
{
    <div class="alert alert-info">@Model.Message</div>
}
```



