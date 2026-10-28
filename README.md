# Travel Logger - Digital Nomad Travel Logging Site (.NET 10)

This branch is identical to `main` except for the .NET version it targets. Use this branch if your cohort is running .NET 10 instead of .NET 8.

## Project Overview

This project is a travel logging site for digital nomads, allowing users to:
- Register and sign in using email as a unique identifier
- Edit their profile  with a description and image URL
- Add logs of cities they've visited, with comments
- Add recommendations for places of interest in cities
- View lists of recommendations and users by city
- Upvote recommendations


### Getting Started

1. Create a new repository using this template. Then each group member should clone that repository.
2. **Important**: Each group member stores the database connection string in their own user secrets (not in `appsettings.Development.json`, which is shared through git). From the project folder, run this with your actual PostgreSQL password:
   ```bash
   dotnet user-secrets set 'TravelLoggerDbConnectionString' 'Host=localhost;Port=5432;Username=postgres;Password=<your_postgresql_password>;Database=TravelLogger'
   ```
   See [User Secrets and Signing In by Email](./AUTH-AND-USER-SECRETS.md) for why, and what to skip from Creek River's setup.
3. Run the following commands to set up the database. The first command, `dotnet restore`, downloads the project's NuGet packages. A fresh clone doesn't have them yet, and `dotnet ef` won't download them for you, so without this step it fails with a `NETSDK1004: Assets file ... not found` error.

```bash
dotnet restore
dotnet ef migrations add InitialCreate
dotnet ef database update
```

4. Start the application:

```bash
dotnet run
```

5. Access the API at `http://localhost:5000`
6. Use Swagger UI at `http://localhost:5000/swagger` to explore and test the API

Note: We've configured the application to use HTTP instead of HTTPS for simplicity during development.

## API Endpoints

### User Endpoints
- `POST /api/users` - Register a new user
- `GET /api/users/signin/{email}` - Sign in a user by email
- `GET /api/users/{id}` - Get user profile, with all their logs and recommendations
- `PUT /api/users/{id}` - Update user profile
- `GET /api/cities/{cityId}/users` - List users by city (based on their most recent log)

### City Endpoints
- `GET /api/cities` - List all cities
- `GET /api/cities/{id}` - Get city details, with logs, users there, and recommendations

### Log Endpoints
- `POST /api/logs` - Create a new log
- `PUT /api/logs/{id}` - Update a log
- `DELETE /api/logs/{id}` - Delete a log
- `GET /api/users/{userId}/logs` - List logs by user
- `GET /api/cities/{cityId}/logs` - List logs by city

### Recommendation Endpoints
- `POST /api/recommendations` - Create a new recommendation
- `PUT /api/recommendations/{id}` - Update a recommendation
- `DELETE /api/recommendations/{id}` - Delete a recommendation
- `GET /api/cities/{cityId}/recommendations` - List recommendations by city
- `GET /api/recommendations/{id}` - Get recommendation details, including total number of upvotes

### Upvote Endpoints
- `POST /api/upvotes` - Add an upvote to a recommendation
- `DELETE /api/upvotes/{id}` - Remove an upvote from a recommendation

## Development Notes
- Begin by considering the data structure, building an Entity Relationship Diagram, and laying out tasks in a Github project board.
- Include all API endpoints in Program.cs. Yes, this will cause conflicts to resolve in GIT as you divide tasks and work independently. Yes, this is intentional, to put you through the ringer practicing GIT workflow with proper pull requests.
- Entity Framework Core should be used for database operations, and the database should be seeded with initial data for cities, users, logs, recommendations, and upvotes
- DTOs should be used to limit the properties that are sent and received
- Authentication should just use email as a unique identifier. Make sure users can't edit or delete other users' data. [User Secrets and Signing In by Email](./AUTH-AND-USER-SECRETS.md) walks through how to do this with what you've learned so far.
