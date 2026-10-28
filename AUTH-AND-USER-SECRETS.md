# User Secrets and Signing In by Email

This project asks for two things you haven't had to combine yet:

1. Keeping the database connection string out of a repository that your whole team shares.
2. "Signing in" users and making sure they can only change their own data.

Neither one needs anything new to install. You'll use `dotnet user-secrets` (from Creek River) and the same `SingleOrDefault`, `Any`, and `Results` methods you already know.

## If your group already started

If your group's repository was created before this document existed, your README still says to put your password in `appsettings.Development.json`. Switch over like this:

1. **One person** removes the `TravelLoggerDbConnectionString` line from `appsettings.Development.json`, and makes sure the file is still valid JSON. If that line came after another setting, delete the comma at the end of the line above it. The file should look like this:
    ``` json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      }
    }
    ```
    Open a pull request with just that change and merge it. Doing it once, in one PR, keeps everyone from getting a merge conflict on the same file.
1. **Everyone** pulls the merged change, then follows [Set it up](#set-it-up-every-teammate-on-their-own-computer) below to store the connection string in their own user secrets.
1. **If anyone pushed their real PostgreSQL password**, that person should change it. Deleting the line doesn't remove it from GitHub, because it's still in the repository's commit history. To change it, open the Query Tool in pgAdmin and run:
    ``` sql
    ALTER USER postgres WITH PASSWORD 'your_new_password';
    ```
    Then use the new password in your `dotnet user-secrets set` command. Your other projects (Creek River, Hillary's Hair Care, and so on) still have the old password in their user secrets, so run `dotnet user-secrets set` again in each one you still use, with that project's own key name.

## Part 1: Store the connection string in user secrets

### Why not `appsettings.Development.json`?

`appsettings.Development.json` is committed to git. If each teammate types their own PostgreSQL password into it:
- Everyone's password ends up on GitHub.
- Your passwords are probably different, so whoever pushed last breaks the connection for everyone else.
- The file shows up in merge conflicts over and over.

User secrets fix all three. Each teammate's secret is saved on their own computer, outside the project folder, so git never sees it.

### Set it up (every teammate, on their own computer)

1. Open a terminal in the project folder (the one with `TravelLogger.csproj` in it).
1. Run this, replacing `<your_postgresql_password>` with your own PostgreSQL password:
    ``` bash
    dotnet user-secrets set 'TravelLoggerDbConnectionString' 'Host=localhost;Port=5432;Username=postgres;Password=<your_postgresql_password>;Database=TravelLogger'
    ```
1. Check that it saved:
    ``` bash
    dotnet user-secrets list
    ```
    You should see `TravelLoggerDbConnectionString = Host=localhost;...` with your password in it.

That's it. A few things are different from Creek River:

- **Skip `dotnet user-secrets init`.** This template already has a `UserSecretsId` in `TravelLogger.csproj`, so it's ready to store secrets. Your whole team shares that same id, which is fine: each computer keeps its own separate secrets file, so nobody's password ever reaches anyone else.
- **Don't change the key name.** `Program.cs` already reads the connection string with `builder.Configuration["TravelLoggerDbConnectionString"]`, so the key you set has to be exactly `TravelLoggerDbConnectionString`.
- **Don't put the connection string back in `appsettings.Development.json`.** If something isn't connecting, run `dotnet user-secrets list` and check the password there.

## Part 2: Signing in by email

### What "auth" means in this project

There are two words you'll hear a lot in Book 4:

**Authentication**: proving you are who you say you are.

**Authorization**: proving you are allowed to do what you're trying to do.

In Book 4 you'll do real authentication, with passwords and cookies. This project is a simpler version you can build with what you know now:
- **Signing in** just means looking up a user by their email. There's no password.
- **Authorization** means each request that changes data says which user is making it, and the API checks that the data belongs to that user.

This is an honor system. Anyone could type someone else's email or user id, and real apps don't work this way. Book 4 replaces it with cookies so the API can tell who is really logged in. For now, the goal is to practice the *checks*, which work the same way once you have real auth.

> :bulb: The code below uses example names (`User`, `Log`, `db.Users`, `db.Logs`). Use whatever names your team chose in your ERD and `TravelLoggerDbContext`.

### Registering: emails must be unique

Since email is how users sign in, two users can't share one. Before adding a new user, check whether that email is already taken:

``` csharp
app.MapPost("/api/users", (TravelLoggerDbContext db, User user) =>
{
    if (db.Users.Any(u => u.Email == user.Email))
    {
        return Results.BadRequest("A user with that email already exists");
    }

    db.Users.Add(user);
    db.SaveChanges();
    return Results.Created($"/api/users/{user.Id}", user);
});
```

### Signing in: look the user up by email

``` csharp
app.MapGet("/api/users/signin/{email}", (TravelLoggerDbContext db, string email) =>
{
    User user = db.Users.SingleOrDefault(u => u.Email == email);
    if (user == null)
    {
        return Results.NotFound();
    }

    return Results.Ok(new UserDTO
    {
        Id = user.Id,
        Name = user.Name,
        Email = user.Email
    });
});
```

The important part of this response is the user's `Id`. A front end would save it after signing in, the same way you saved the signed-in user in the front-end course, and send it along with every request that changes data. In Swagger, you'll just copy it and paste it into the next request yourself.

### Protecting edits and deletes

Every endpoint that **updates** or **deletes** something needs to know who is asking. Add an `int userId` parameter to the handler. Because `userId` isn't part of the route (`{id}` is), ASP.NET reads it from the *query string* at the end of the URL:

```
PUT /api/logs/5?userId=2
```

Then, after finding the row, compare its owner to `userId`:

``` csharp
app.MapPut("/api/logs/{id}", (TravelLoggerDbContext db, int id, int userId, Log log) =>
{
    Log logToUpdate = db.Logs.SingleOrDefault(l => l.Id == id);
    if (logToUpdate == null)
    {
        return Results.NotFound();
    }
    if (logToUpdate.UserId != userId)
    {
        return Results.StatusCode(403);
    }

    logToUpdate.Comment = log.Comment;
    db.SaveChanges();
    return Results.NoContent();
});

app.MapDelete("/api/logs/{id}", (TravelLoggerDbContext db, int id, int userId) =>
{
    Log log = db.Logs.SingleOrDefault(l => l.Id == id);
    if (log == null)
    {
        return Results.NotFound();
    }
    if (log.UserId != userId)
    {
        return Results.StatusCode(403);
    }

    db.Logs.Remove(log);
    db.SaveChanges();
    return Results.NoContent();
});
```

A few things to notice:
- **`403` means "Forbidden".** The request was understood, but this user isn't allowed to do it. You might find `Results.Forbid()` and try it instead. Don't: it only works once real authentication is set up (Book 4), and in this project it crashes the request with a `500` error.
- **Check "not found" first, then ownership.** You can't compare owners until you know the row exists.
- **The owner comes from the database, not the request body.** Compare `logToUpdate.UserId` (what's saved) against `userId` (who's asking). Don't let an update change `UserId` either, or a user could hand their log to someone else.

The user profile endpoint is the same idea, except the thing being edited *is* the user, so compare the route `id` to `userId` directly:

``` csharp
app.MapPut("/api/users/{id}", (TravelLoggerDbContext db, int id, int userId, User user) =>
{
    if (id != userId)
    {
        return Results.StatusCode(403);
    }

    User userToUpdate = db.Users.SingleOrDefault(u => u.Id == id);
    if (userToUpdate == null)
    {
        return Results.NotFound();
    }

    userToUpdate.Description = user.Description;
    userToUpdate.ImageUrl = user.ImageUrl;
    db.SaveChanges();
    return Results.NoContent();
});
```

### Which endpoints need the check

| Endpoint | Allowed when |
|---|---|
| `PUT /api/users/{id}` | `id` is the signed-in user |
| `PUT /api/logs/{id}` and `DELETE /api/logs/{id}` | the log's `UserId` is the signed-in user |
| `PUT /api/recommendations/{id}` and `DELETE /api/recommendations/{id}` | the recommendation's `UserId` is the signed-in user |
| `DELETE /api/upvotes/{id}` | the upvote's `UserId` is the signed-in user |

`GET` endpoints don't need it, because anyone can view logs, recommendations, and profiles. For `POST` endpoints (creating a log, recommendation, or upvote), the new row's `UserId` should be the signed-in user's id, which the front end sends in the request body.

### Testing it in Swagger

1. Use `POST /api/users` to register a user (or use one from your seed data), then sign in with `GET /api/users/signin/{email}`. Note the `id` in the response.
1. Try to register the same email again. You should get `400`.
1. Open `PUT /api/logs/{id}` or `DELETE /api/logs/{id}`. Swagger shows a separate `userId` box for the query string.
    - Use a log that belongs to your user and your user's id: you should get `204`.
    - Use the same log with a *different* user's id: you should get `403`, and nothing in the database should change.
