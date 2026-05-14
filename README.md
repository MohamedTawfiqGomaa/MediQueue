# MediQueue

A cross-platform dental patient management application built with .NET MAUI, ASP.NET Core, and PostgreSQL.

## Features

- **Patient Management**: CRUD operations for patient records
- **Appointment Scheduling**: Manage patient appointments
- **Treatment Planning**: Create and track treatment plans
- **Dental Charting**: Visual representation of dental conditions
- **User Authentication**: Secure login and role-based access control
- **Data Synchronization**: Real-time data updates between client and server
- **Cross-Platform**: Runs on Windows, macOS, and Linux
- **Progressive Web App**: Installable on mobile devices

## Getting Started

### Prerequisites

- **.NET SDK 10.0** (or higher)
- **ASP.NET Core Runtime** (required for server)
- **PostgreSQL** (database server)

### Installation

1. **Clone the repository**

```bash
git clone <repository-url>
cd MediQueue
```

2. **Restore dependencies**

```bash
dotnet restore
```

3. **Configure the database**

Ensure you have a PostgreSQL database running and update the connection string in:
- `MediQueue.Client/appsettings.json`
- `MediQueue.Server/appsettings.json`

4. **Run the application**

```bash
dotnet run --project MediQueue.Server
```

5. **Access the app**

Open your browser and navigate to:
- **Client App**: http://localhost:5000
- **Admin Dashboard**: http://localhost:5001

## Project Structure

```
MediQueue/
├── MediQueue.Client/          # .NET MAUI client application
├── MediQueue.Server/          # ASP.NET Core server application
├── MediQueue.Shared/         # Shared code and types
├── MediQueue.Worker/          # Background worker service
└── MediQueue.sln              # Solution file
```

## Tech Stack

### Client
- [.NET MAUI](https://dotnet.microsoft.com/apps/maui)
- C#
- XAML

### Server
- [ASP.NET Core 10.0](https://dotnet.microsoft.com/apps/aspnet)
- [Entity Framework Core](https://docs.microsoft.com/ef/core/)
- [JWT Authentication](https://docs.microsoft.com/aspnet/core/security/authentication/jwt-auth)
- [AutoMapper](https://docs.automapper.org/)

### Database
- PostgreSQL

### Additional Tools
- [FluentValidation](https://docs.fluentvalidation.net/)
- [AutoMapper](https://docs.automapper.org/)
- [Serilog](https://serilog.net/)
- [Swashbuckle](https://docs.microsoft.com/aspnet/core/tutorials/make-api-with-swagger)

## Development

### Running Migrations

To apply database migrations:

```bash
dotnet ef migrations add <MigrationName> --project MediQueue.Server --startup-project MediQueue.Server
dotnet ef database update --project MediQueue.Server --startup-project MediQueue.Server
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please read the [CONTRIBUTING.md](CONTRIBUTING.md) file for more information.

## Contact

For questions or support, please open an issue or contact the development team.

## Acknowledgments

- .NET Community
- ASP.NET Core Team
- MAUI Community
- PostgreSQL Community

---

*Built with ❤️ using modern .NET technologies*
# MediQueue
# MediQueue
