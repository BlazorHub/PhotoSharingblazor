# FrontEnd: Comments

If you followed Lab06, you can continue from your own project. Otherwise, open the `Lab07/Start` solution and continue from there.

---

Now that we took care of the Photos, we will proceed to implement the Comments functionality. Our `Details` Page will allow the user to 

- See the Comments related to the Photo
- Post a  new Comment
- Update an existing Comment
- Delete an existing Comment

We will also create a first C# Data Layer with methods to 
- get a list of all the comments for a given photo
- get one comment given its id
- create a given comment
- update a given comment
- delete a comment given its id 

For now, our data layer will be a prototype that works with a List in memory, since we want to focus on the UI.
We will replace it with one that can communicate with a gRpc service in a later lab, when we start thinking about the backend.

We're going to continue with the [CLEAN architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) that we already have.

- In The *Shared* project we will introduce the *Comment* entity and the interfaces for the *CommentsService* and the *CommentsRepository*
- In the *Core* project we will define the business logic for the *CommentsService*
- In the *Infrastructure* project we will define a *CommentsRepository*. For now we have a simple Repository class that uses a List. Later we'll talk to a gRpc service

## The Shared Code

We are going to define two interfaces: one for an ICommentsService and one for an ICommentsRepository. Our interfaces will look very similar and will contain the definitions for the method to Create, Read, Update and Delete photos. Both are going to use Comment entities, that we also have to define in this project. 

### The Comment Entity

- In the `Shared`, under the `Entities` folder, add the following `Comment` class:

```cs
using System;

namespace PhotoSharingApplication.Shared.Core.Entities {
  public class Comment {
    public int Id {get; set; }   
    public int PhotoId { get; set; }
    public string UserName { get; set; }
    public string Subject { get; set; }
    public string Body { get; set; }
    public DateTime SubmittedOn { get; set; }
    public Photo Photo { get; set; }
  }
}
```

Also, add a new navigation property to the `Photo` class:

```cs
public virtual ICollection<Comment> Comments { get; set; }
```

### The Interfaces

- Under the `Interfaces` folder, add the following `ICommentsService` interface:

```cs
using PhotoSharingApplication.Shared.Core.Entities;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace PhotoSharingApplication.Shared.Core.Interfaces {
  public interface ICommentsService {
    Task<List<Comment>> GetCommentsForPhotoAsync(int photoId);
    Task<Comment> FindAsync(int id);
    Task<Comment> CreateAsync(Comment comment);
    Task<Comment> UpdateAsync(Comment comment);
    Task<Comment> RemoveAsync(int id);
  }
}
```

- Under the same folder, add the following `ICommentsRepository` interface

```cs
using PhotoSharingApplication.Shared.Core.Entities;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace PhotoSharingApplication.Shared.Core.Interfaces {
  public interface ICommentsRepository {
    Task<List<Comment>> GetCommentsForPhotoAsync(int photoId);
    Task<Comment> FindAsync(int id);
    Task<Comment> CreateAsync(Comment comment);
    Task<Comment> UpdateAsync(Comment comment);
    Task<Comment> RemoveAsync(int id);
  }
}
```

## The Frontend Core

`PhotoSharingApplication.Frontend.Core`

Now we can implement our service, which for now will just pass the data to the repository and return the results, without any additional logic (we will replace it later). We are going to use the [Dependency Injection pattern](https://martinfowler.com/articles/injection.html?) to request for a repository.

```cs
using PhotoSharingApplication.Shared.Core.Entities;
using PhotoSharingApplication.Shared.Core.Interfaces;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace PhotoSharingApplication.Frontend.Core.Services {
  public class CommentsService : ICommentsService {
    private readonly ICommentsRepository repository;

    public CommentsService(ICommentsRepository repository) {
      this.repository = repository;
    }

    public async Task<Comment> CreateAsync(Comment comment) => await repository.CreateAsync(comment);

    public async Task<Comment> FindAsync(int id) => await repository.FindAsync(id);

    public async Task<List<Comment>> GetCommentsForPhotoAsync(int photoId) => await repository.GetCommentsForPhotoAsync(photoId);

    public async Task<Comment> RemoveAsync(int id) => await repository.RemoveAsync(id);

    public async Task<Comment> UpdateAsync(Comment comment) {
      comment.SubmittedOn = DateTime.Now;
      return await repository.UpdateAsync(comment);
    }
  }
}

```

Of course nothing is actually *working*, but we can already start plugging our service to our UI.

- On the `Solution Explorer`, right click on the `Dependencies` folder of the `PhotoSharingApplication.Frontend.BlazorWebAssembly` project

To use our service in the `Details` page, we need to perform a couple of steps, also described in the [Blazor Dependency Injection documentation](https://docs.microsoft.com/en-gb/aspnet/core/blazor/dependency-injection?view=aspnetcore-5.0)

### Add the service to the app

[In the docs](https://docs.microsoft.com/en-gb/aspnet/core/blazor/dependency-injection?view=aspnetcore-5.0#add-services-to-an-app) they tell us what to do: 

- Open the `Program.cs` file of the `PhotoSharingApplication.Frontend.BlazorWebAssembly`project
- Add the following code, before the `await builder.Build().RunAsync();`

```cs
builder.Services.AddScoped<ICommentsService, CommentsService>();
```

## The Frontend Infrastructure

- In the `Memory` folder, of the `PhotoSharingApplication.Frontend.Infrastructure` project, add a new class `CommentsRepository` with the following code

```cs
using PhotoSharingApplication.Shared.Core.Entities;
using PhotoSharingApplication.Shared.Core.Interfaces;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace PhotoSharingApplication.Frontend.Infrastructure.Repositories.Memory {
  public class CommentsRepository : ICommentsRepository {
    private List<Comment> comments;
    public CommentsRepository() {
      comments = new List<Comment>() { 
        new Comment(){ Id=1, Subject="A Comment", Body="The Body of the comment", SubmittedOn=DateTime.Now.AddDays(-1), PhotoId=1 },
        new Comment(){ Id=2, Subject="Another Comment", Body="Another Body of the comment", SubmittedOn=DateTime.Now.AddDays(-2), PhotoId=1 },
        new Comment(){ Id=3, Subject="Yet another Comment", Body="Yet Another Body of the comment", SubmittedOn=DateTime.Now, PhotoId=2 },
        new Comment(){ Id=4, Subject="More Comment", Body="More Body of the comment", SubmittedOn=DateTime.Now.AddDays(-3), PhotoId=2 }
      };
    }
    public Task<Comment> CreateAsync(Comment comment) {
      comment.Id = comments.Max(p => p.Id) + 1;
      comments.Add(comment);
      return Task.FromResult(comment);
    }

    public Task<Comment> FindAsync(int id) => Task.FromResult(comments.FirstOrDefault(p => p.Id == id));


    public Task<List<Comment>> GetCommentsForPhotoAsync(int photoId) => Task.FromResult(comments.Where(c=>c.PhotoId==photoId).OrderByDescending(c => c.SubmittedOn).ThenBy(c => c.Subject).ToList());

    public Task<Comment> RemoveAsync(int id) {
      Comment comment = comments.FirstOrDefault(c => c.Id == id);
      if (comment!= null) comments.Remove(comment);
      return Task.FromResult(comment);
    }

    public Task<Comment> UpdateAsync(Comment comment) {
      Comment oldComment = comments.FirstOrDefault(c => c.Id == comment.Id);
      if (oldComment != null) {
          oldComment.Subject = comment.Subject;
          oldComment.Body= comment.Body;
      }
      return Task.FromResult(oldComment);
    }
  }
}
```

I know, I know, it's a very naive implementation, but it's just to have something working so that we can see some action in the UI, we're going to replace it with something better later anyway.

Our last step is to plug this implementation in our application, so that the CommentsService can use it. We do this simply by registering this class as a service during startup.

- Open the `Program.cs` file of the `PhotoSharingApplication.Frontend.BlazorWebAssembly`project
- Add the following code, before the `await builder.Build().RunAsync();`

```cs
builder.Services.AddScoped<ICommentsRepository, PhotoSharingApplication.Frontend.Infrastructure.Repositories.Memory.CommentsRepository>();
```

## The Comments UI

We want to enrich the `PhotoDetails` Page by 

- Retreiving the comments of a `Photo`, by talking to the `CommentsService` 
- Displaying the details of each comment 
- Displaying a form to submit a new comment

We could do this directly on the `PhotoDetails` Page, but it would start getting a bit too crowded and hard to maintain.
We shouldn't give too many responsibilities to the `PhotoDetails` page.

We can split the functionalities by creating a `CommentsComponent` and referring to it from within the `PhotoDetails` page.

The `CommentsComponent` will receive the Id of the Photo and take care of the rest. The only thing we need to add to the `Detals` page is

```html
<div class="col-sm-12">
  @if (photo == null)
  {
    <p><em>Loading...</em></p>
  }
  else
  {
    <Photo PhotoItem="photo" Edit="mayEditDeletePhoto" Delete="mayEditDeletePhoto" OnDelete="OnDeletePhoto" OnEdit="OnEditPhoto"></Photo>
    <CommentsComponent PhotoId="Id"></CommentsComponent>
  }
</div>
```

## The CommentsComponent

Let's put the `CommentsComponent` in the `Shared` folder of the `PhotoSharingApplication.Frontend.BlazorWebAssembly` project.

Here, we will provide a `[Parameter]` to get the `PhotoId`.

```cs
@code {
  [Parameter]
  public int PhotoId { get; set; }
}
```

We will also get the dependency on the `CommentsService` and initialize a `List` of comments.

```cs
@using PhotoSharingApplication.Shared.Core.Interfaces
@using PhotoSharingApplication.Shared.Core.Entities

@inject ICommentsService CommentsService

<h3>CommentsComponent</h3>

@code {
  [Parameter]
  public int PhotoId { get; set; }

  private List<Comment> comments;

  protected override async Task OnInitializedAsync() {
    comments = await CommentsService.GetCommentsForPhotoAsync(PhotoId);
  }
}
```

If there are comments, we will scroll through the list and render the details of each comment. We will also display a form to submit a new comment.

Once again, we don't want to have this bit too crowded, so let's think about a `Comment` component capable of changing its own appearance in place.

Our `CommentComponent` will have a `Property` *`ViewMode`* with four different possible states:

- Read
- Edit
- Delete
- Create

Depending on the value of this property, the component will render specific HTML for specific functionalities. In each mode we will have buttons to switch to a different mode if necessary. 

Let's first use it from our `CommentsComponent`, which becomes:

```html
<h3>CommentsComponent</h3>

@if (comments == null) {
  <p><em>No Comments for this Photo</em></p>
} else {
  <div class="list-group">
    @foreach (var comment in comments) {
      <CommentComponent CommentItem="comment" ViewMode="CommentComponent.ViewModes.Read"></CommentComponent>
    }
      <CommentComponent CommentItem="new Comment() {PhotoId = PhotoId}" ViewMode="CommentComponent.ViewModes.Create"></CommentComponent>
    }
  </div>
}
```

## The CommentComponent

We can create the `CommentComponent` in the `PhotoSharingApplication.Frontend.BlazorComponents` project.

We will accept a `[Parameter]` for the `Comment` and a `[Parameter]` for the *ViewMode*. The `ViewMode` parameter will be of a new `enum` type that we will define in the component.

```cs
@using PhotoSharingApplication.Shared.Core.Entities
<h3>CommentComponent</h3>

@code {
  [Parameter]
  public Comment CommentItem { get; set; }
  
  [Parameter]
  public ViewModes ViewMode { get; set; }

  public enum ViewModes { 
    Read, Edit, Delete, Create
  }
}
```

Our `CommentComponent` will render a different subcomponent depending on the `ViewMode`:

```html
<MatCard>
@if (ViewMode == ViewModes.Read) {
  <CommentReadComponent CommentItem="CommentItem"></CommentReadComponent>
} else if (ViewMode == ViewModes.Edit) {
  <CommentEditComponent CommentItem="CommentItem"></CommentEditComponent>
} else if (ViewMode == ViewModes.Delete) {
  <CommentDeleteComponent CommentItem="CommentItem"></CommentDeleteComponent>
} else if (ViewMode == ViewModes.Create) {
  <CommentCreateComponent CommentItem="CommentItem"></CommentCreateComponent>
}
</MatCard>
```

Each of these component will render the correct HTML and  it will  raise events that we will handle here, either to switch view or to let the page know that it's time to talk to the service to create / update / delete a comment.

```html
<MatCard>
@if (ViewMode == ViewModes.Read) {
  <CommentReadComponent CommentItem="CommentItem" OnEdit="SwitchToEditMode" OnDelete="SwitchToDeleteMode"></CommentReadComponent>
} else if (ViewMode == ViewModes.Edit) {
  <CommentEditComponent CommentItem="CommentItem" OnSave="ConfirmUpdate" OnCancel="CancelUpdate"></CommentEditComponent>
} else if (ViewMode == ViewModes.Delete) {
  <CommentDeleteComponent CommentItem="CommentItem" OnDelete="ConfirmDelete" OnCancel="SwitchToReadMode"></CommentDeleteComponent>
} else if (ViewMode == ViewModes.Create) {
  <CommentCreateComponent CommentItem="CommentItem" OnSave="ConfirmCreate"></CommentCreateComponent>
}
</MatCard>
```

The code to handle these events will switch ViewMode eventually after notifying the parent component. We will also need some `EventCallback` to notify the parent component.

The code to switch view mode is fairly simple:

```cs
void SwitchToReadMode() => ViewMode = ViewModes.Read;
void SwitchToEditMode() => ViewMode = ViewModes.Edit;
void SwitchToDeleteMode() => ViewMode = ViewModes.Delete;
```

The `ConfirmUpdate` will notify the parent component then switch to read mode:

```cs
[Parameter]
public EventCallback<Comment> OnUpdate { get; set; }

async Task ConfirmUpdate() {
    await OnUpdate.InvokeAsync(CommentItem);
    SwitchToReadMode();
}
```

The `CancelUpdate` will be a sort of *undo*, because the user may have changed the values of our `CommentItem` by writing on some textfields.
This means we need to store a copy of our CommentItem and restore the values on cancel.

```cs
private Comment originalComment;

protected override void OnInitialized() {
  originalComment = new Comment { Subject = CommentItem.Subject, Body = CommentItem.Body };
}

public void CancelUpdate() {
  CommentItem.Subject = originalComment.Subject;
  CommentItem.Body = originalComment.Body;

  SwitchToReadMode();
}
```

The `ConfirmDelete` will notify the parent component then switch to read mode: 

```cs
[Parameter]
public EventCallback<Comment> OnDelete{ get; set; }

async Task ConfirmDelete() {
  await OnDelete.InvokeAsync(CommentItem);
  SwitchToReadMode();
}
```

The `ConfirmCreate` will notify the parent component then create a new comment to reset the fields.

```cs
[Parameter]
public EventCallback<Comment> OnCreate { get; set; }

async Task ConfirmCreate() {
  await OnCreate.InvokeAsync(CommentItem);
  CommentItem = new Comment() { PhotoId = CommentItem.PhotoId, UserName = CommentItem.UserName };
}
```

That's all for this component, so let's create the four sub components.

## The CommentReadComponent

This component shows a `Card` with the details of the comment and two buttons to switch to Edit and Delete mode.
Its logic only notifies its parent component.

```cs
@using PhotoSharingApplication.Shared.Core.Entities 

<MatCardContent>
  <em>On @CommentItem.SubmittedOn.ToShortDateString() At @CommentItem.SubmittedOn.ToShortTimeString(), @CommentItem.UserName said:</em>
  <MatH5>@CommentItem.Subject</MatH5>
  <p>@CommentItem.Body</p>
</MatCardContent>
<MatCardActions>        
  <MatButton OnClick="RaiseEdit">Edit</MatButton>
  <MatButton OnClick="RaiseDelete">Delete</MatButton>
</MatCardActions>

@code {
  [Parameter]
  public Comment CommentItem { get; set; }

  [Parameter]
  public EventCallback<Comment> OnEdit { get; set; }

  [Parameter]
  public EventCallback<Comment> OnDelete { get; set; }

  async Task RaiseEdit(MouseEventArgs args) => await OnEdit.InvokeAsync(CommentItem);
  async Task RaiseDelete(MouseEventArgs args) => await OnDelete.InvokeAsync(CommentItem);
  }

```

## The CommentEditComponent

This component will have an `EditForm` with two TextFields (for `Subject` and `Body`) and two buttons (for `Update` and `Cancel`).
The logic will notify the parent component through the use of event callbacks.

```cs
@using PhotoSharingApplication.Shared.Core.Entities 

<MatCardContent>
  <EditForm Model="@CommentItem" OnValidSubmit="HandleValidSubmit">
    <p>
      <MatTextField @bind-Value="@CommentItem.Subject" Label="Subject" FullWidth></MatTextField>
    </p>
    <p>
      <MatTextField @bind-Value="@CommentItem.Body" Label="Description" TextArea FullWidth></MatTextField>
    </p>
    <p>
      <MatButton Type="submit">Update</MatButton>
      <MatButton OnClick="OnCancel">Cancel</MatButton>
    </p>
  </EditForm>
</MatCardContent>

@code {
  [Parameter]
  public Comment CommentItem { get; set; }

  [Parameter]
  public EventCallback OnCancel { get; set; }

  [Parameter]
  public EventCallback<Comment> OnSave { get; set; }

  private async Task HandleValidSubmit() {
    await OnSave.InvokeAsync(CommentItem);
  }
}
```
## The CommentDeleteComponent

The `CommentDeleteComponent` will show the details of the comment and it will provide two buttons to confirm deletion and to cancel. Its logic will notify the parent component through `EventCallback`s.

```cs
@using PhotoSharingApplication.Shared.Core.Entities

<MatCardContent>
  <MatCaption Class="mat-text-danger">Are you sure you want to delete this comment?</MatCaption>
  <em>On @CommentItem.SubmittedOn.ToShortDateString() At @CommentItem.SubmittedOn.ToShortTimeString(), @CommentItem.UserName said:</em>
  <MatH5>@CommentItem.Subject</MatH5>
  <p>@CommentItem.Body</p>
</MatCardContent>
<MatCardActions>
  <MatButton OnClick="OnCancel">Cancel</MatButton>
  <MatButton OnClick="OnDelete">Delete</MatButton>
</MatCardActions>

@code {
  [Parameter]
  public Comment CommentItem { get; set; }

  [Parameter]
  public EventCallback OnCancel { get; set; }

  [Parameter]
  public EventCallback OnDelete { get; set; }
}
```
## The CommentCreateComponent

The `CommentCreateComponent` will have an `EditForm` with two TextFields (for `Subject` and `Body`) and one button for `Save`.
The logic will notify the parent component through the use of an `EventCallback`.

```cs
@using PhotoSharingApplication.Shared.Core.Entities

<MatCardContent>
  <EditForm Model="@CommentItem" OnValidSubmit="HandleValidSubmit">
    <p>
      <MatTextField @bind-Value="@CommentItem.Subject" Label="Subject" FullWidth></MatTextField>
    </p>
    <p>
      <MatTextField @bind-Value="@CommentItem.Body" Label="Description" TextArea FullWidth></MatTextField>
    </p>
    <p>
      <MatButton Type="submit">Submit</MatButton>
    </p>
  </EditForm>
</MatCardContent>

@code {
  [Parameter]
  public Comment CommentItem { get; set; }

  [Parameter]
  public EventCallback<Comment> OnSave { get; set; }

  private async Task HandleValidSubmit() {
      await OnSave.InvokeAsync(CommentItem);
  }
}
```

The last thing we need to do is to handle the actual `OnCreate`, `OnUpdate` and `OnDelete` events in the `CommentsComponent` to actually invoke the methods of the `CommentsService`.

Open the `CommentsComponent` and replace its content with the following code:


```cs
@using PhotoSharingApplication.Shared.Core.Interfaces
@using PhotoSharingApplication.Shared.Core.Entities

@inject ICommentsService CommentsService

<h3>CommentsComponent</h3>

@if (comments == null) {
  <p><em>No Comments for this Photo</em></p>
} else {
  <div class="list-group">
    @foreach (var comment in comments) {
      <CommentComponent CommentItem="comment" ViewMode="CommentComponent.ViewModes.Read" OnUpdate="UpdateComment"  OnDelete="DeleteComment"></CommentComponent>
    }
    <CommentComponent CommentItem="new Comment() {PhotoId = PhotoId}" ViewMode="CommentComponent.ViewModes.Create" OnCreate="CreateComment"></CommentComponent>
  </div>
}
@code {
  [Parameter]
  public int PhotoId { get; set; }

  private List<Comment> comments;

  protected override async Task OnInitializedAsync() {
    comments = await CommentsService.GetCommentsForPhotoAsync(PhotoId);
  }

  async Task CreateComment(Comment comment) {
    comments.Add(await CommentsService.CreateAsync(comment));
  }

  async Task UpdateComment(Comment comment) {
    comment = await CommentsService.UpdateAsync(comment);
  }

  async Task DeleteComment(Comment comment) {
    await CommentsService.RemoveAsync(comment.Id);
    comments.Remove(comment);
  }
}
```

We did change the `Photo` model, so if we run the application now the server side complains that the database does not match the model.

Just to test if our frontend works, we're going to temporarily switch the `PhotosRepository` with the old `Memory` one.

This way we stay client side so that we don't run into problems when talking to the server side, which we will fix in the following lab.

- Open `Program.cs` of the `PhotoSharingApplication.Frontend.BlazorWebAssembly` project 
- Comment the following line

```cs
// builder.Services.AddScoped<IPhotosRepository, PhotoSharingApplication.Frontend.Infrastructure.Repositories.Rest.PhotosRepository>();
```
- Add the following line
```cs
builder.Services.AddScoped<IPhotosRepository, PhotoSharingApplication.Frontend.Infrastructure.Repositories.Memory.PhotosRepository>();
```

If you run the application now and navigate to `/photos/details/1` you should see the comments under the Photo details.

Clicking on the different buttons should switch view and perform the relative actions.

That's it for this lab, our frontend is ready for now.

---

In the following lab, we're taking care of the backend. We will build the functionalities to save the comments in the database and we will serve them through a gRpc Service.

Go to `Labs/Lab08`, open the `readme.md` and follow the instructions thereby contained.