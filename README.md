# Boot Camp Live Projects


## Introduction

I did a two week internship at Prosper IT Consulting working on an an ASP.NET MVC site for a local theater group. It being Covid-time, we worked remotely. We used Slack, Google Meet, and daily scrum meetings to maintain group cohesion. I was able to work on several front end and back end stories.

Back End:

-[Delete Users](README.md#delete-users)

-[Admin Overlay](README.md#admin-overlay)


Front End:

-[Edit Users](README.md#edit-users)

-[Templates View](README.md#templates-view)

-[Form Bug Fix](README.md#form-bug-fix)

### Delete users

This story task was to add the ability to delete users from the admin page, while preserving any sorts or filters used on the page. It was a fairly straightforward task with the main hurdle being familiarizing myself with the existing user management system and the database. The implementation on the front end was just a delete button and a confirmation modal.

```c#
/// <summary>
/// Delete a User from the Admin/UserList
/// </summary>
/// <param name="id">User Id</param>
/// <returns>The referring View path</returns>
[HttpPost, ActionName("DeleteUser")]
[ValidateAntiForgeryToken]
public async Task<ActionResult> UserList(string id)
{
	var userManager = new UserManager<ApplicationUser>(new UserStore<ApplicationUser>(db));

	if (ModelState.IsValid)
	{
		//make sure we got an id
		if (id == null)
		{
			return new HttpStatusCodeResult(HttpStatusCode.BadRequest);
		}

		var userToBeDeleted = await userManager.FindByIdAsync(id);
		var userRolesToBeDeleted = await userManager.GetRolesAsync(id);
		//open a connection to the database context, start new transaction
		using (var transaction = db.Database.BeginTransaction()) 
		{
			if (userToBeDeleted.Role == "Subscriber") //if user is a subscriber, remove that data, otherwise fk errors
			{
				var subdata = db.Subscribers.Where(x => id.Equals(id)); //get all the rows with ids that match ours from dbo.Subscribers
				db.Subscribers.RemoveRange(subdata); //delete them
			}
			if (userRolesToBeDeleted.Count() > 0) //if user has roles, get rid of them
			{
				foreach (var item in userRolesToBeDeleted.ToList())
				{
					await userManager.RemoveFromRoleAsync(userToBeDeleted.Id, item);
				}
			}
			await userManager.DeleteAsync(userToBeDeleted); //delete the user
			transaction.Commit(); //commit and close down the connection
		}
		return Redirect(Request.UrlReferrer.PathAndQuery); //if it all worked out, go back to admin/userlist and keep the same sort/filters/search
	}
	else
	{
		return new HttpStatusCodeResult(HttpStatusCode.BadRequest); //something went wrong, invalid model
	}
}
```

### Admin Overlay

While building the app, we had an admin overlay for reporting bugs. My task for this story was to make the opened/closed state of the tab persist through navigation. I did this with an admin controller method that would write the state of the tab to an already existing admin settings json file.

```c#
/// <summary>
/// Stores the state of the Bug Report widget in AdminSettings.json
/// </summary>
/// <param name="bugreport">BugReportTab object</param>
/// <returns></returns>
[HttpPost]
public ActionResult BugTabStateUpdate(BugReportTab bugreport)
{
	string filepath = Server.MapPath(Url.Content("~/AdminSettings.json"));

	//Get the current settings
	AdminSettings currentAdminSettings = AdminSettingsReader.CurrentSettings();

	//Insert our new value
	currentAdminSettings.BugReport = bugreport;

	//Convert to json
	string newJson = JsonConvert.SerializeObject(currentAdminSettings, Formatting.Indented);

	//Write to file
	using (StreamWriter writer = new StreamWriter(filepath))
	{
		writer.Write(newJson);
	}
	//make Ajax happy with a success return
	return Json(new { success = true, message = $"tab_open: {bugreport.tab_open}" });
}
```

I also had to write a JavaScript Ajax function to do the sending of state of the tab to the controller method. A bit of jQuery made this pretty easy.

```javascript
<script>
//Send the state of the bug report tab to the back-end
	function BugTabStateToJson(tabstate) {

	var bugtab = { "tab_open": tabstate.data.tab_open };

	$.ajax({
	type: 'POST',
	url: '@Url.Action("BugTabStateUpdate", "Admin")',
	data: JSON.stringify(bugtab),
	contentType: 'application/json; charset=utf-8',
	dataType: 'json',
	success: function (message) {
		console.log(message);
	},
	error: function (XMLHttpRequest, textStatus, errorThrown) {
		console.log(`Request: ${XMLHttpRequest.toString()}\nStatus: ${textStatus}\nError:${errorThrown}`);
	}
	});
	return false;
}
$("#bug_icon_btn_leftarrow").on("click", { "tab_open": "true" }, BugTabStateToJson);
$("#bug_icon_btn_rightarrow").on("click", { "tab_open": "false" }, BugTabStateToJson);
</script>

```

### Edit Users

This was a front end story. The page that allowed users to edit their own details needed to have its appearance brought in line with other forms on the site. The story for this provided me with a design and I mostly wrote some HTML to make it match those specifications. The only tricky part was a requirement that a default image from the database be displayed if the user had not uploaded one. Or if the user was a cast member, that photo should be used. I accomplished this by adding the following bit of code to the existing controller method.

```c#
//Get CastMember photo if there is one, otherwise use default photo.
using (var db = new ApplicationDbContext())
{
    //get the value from the db, should only be one
    //grab first result, and convert it to int
    //if conversion fails, defaults to the current generic PhotoId 9
    model.PhotoId = (from s in db.CastMembers
                    where s.CastMemberID == user.CastMemberUserID
                    select s.PhotoId).FirstOrDefault().GetValueOrDefault(9);
}
```

### Templates View

For this front end story, I was given the task of creating a new admin view to showcase various site templates for reference by other developers. It was a straightforward view method and some simple HTML. It was my first story in the project and I took it on because it was a good one for scoping out the rest of the web app.

### Form Bug Fix

For this front end story, I was given a story to fix a certain form that crashed the site when visited. The solution ended up that another dev had made a change to the model that form used. The input in question indicated if a theater production was currently showing, the other dev added some logic so that users wouldn't have to supply that info. So the form was trying to get a field that no longer accepted user input and was failing. Once I figured all that out, it was simple to remove the field.
