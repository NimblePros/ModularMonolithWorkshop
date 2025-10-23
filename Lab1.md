# Lab 1: Create a modular monolith solution with one module for Users

In this lab we will create the initial structure of the modular monolith application. You have two options:

1. Create the files from scratch using `dotnet` CLI or your IDE.
2. Create the files using a `dotnet new` template called `Ardalis.Modulith`.

In either case the resulting solution file should be called `Nimble.Modulith` and should contain five projects when completed:

- AspireHost
- ServiceDefaults
- Web (ASP.NET Core app)
- Users
- Users.Contracts

## Option 1: Creating the Solution with dotnet CLI

Follow these steps to create the modular monolith solution from scratch:

### 1. Create the solution file

```bash
dotnet new sln --name Nimble.Modulith --output .
```

This creates a `Nimble.Modulith.slnx` file in the repository root.

### 2. Create the AspireHost project

First, install the Aspire CLI if you haven't already:

```bash
dotnet tool install -g Aspire.Cli --prerelease
```

Verify the tool is installed:

```bash
aspire --version
```
should yield 9.5.x

Then create the AspireHost and ServiceDefaults projects:

```bash
aspire new aspire
```
When asked, give the name Nimble.Modulith. Keep the default output path. No tests.

### 3. Change to slnx file

```bash
cd Nimble.Modulith
dotnet sln migrate
rm .\Nimble.Modulith.sln
```

Now you should have the following folders:

Nimble.Modulith.AppHost
Nimble.Modulith.ServiceDefaults

and files:

Nimble.Modulith.slnx

### 4. Create the ASP.NET Core minimal Web API project

Run the following commands from the same folder as your slnx file.

```bash
dotnet new webapi --name Nimble.Modulith.Web --output Nimble.Modulith.Web --use-minimal-apis --framework net10.0
dotnet sln add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj --solution-folder "_Host"
```

### 5. Create the Users class library

```bash
dotnet new classlib --name Nimble.Modulith.Users --output Nimble.Modulith.Users --framework net10.0
```

### 6. Create solution folder for Users Module

```bash
dotnet sln add Nimble.Modulith.Users/Nimble.Modulith.Users.csproj --solution-folder "Users Module"
```

### 7. Create the Users.Contracts class library

```bash
dotnet new classlib --name Nimble.Modulith.Users.Contracts --output Nimble.Modulith.Users.Contracts --framework net10.0
dotnet sln add Nimble.Modulith.Users.Contracts/Nimble.Modulith.Users.Contracts.csproj --solution-folder "Users Module"
```

### 8. Verify all projects are in the solution file

```bash
dotnet sln list
```

You should see all 5 projects. If not you can use `dotnet sln add <csproj path>` to add any that are missing.

Add the Aspire projects to the `_Host` solution folder:

```bash
dotnet sln add Nimble.Modulith.AppHost/Nimble.Modulith.AppHost.csproj --solution-folder "_Host"
dotnet sln add Nimble.Modulith.ServiceDefaults/Nimble.Modulith.ServiceDefaults.csproj --solution-folder "_Host"
```

### 9. Ensure all projects target .NET 10

The Aspire projects created by `aspire new` may target a different framework version. Use a central `Directory.Build.props` file to set the .NET version and remove the `<TargetFramework>net9.0</TargetFramework>` element from the two Aspire projects.


```bash
dotnet new buildprops
```

Edit the `Directory.Build.props` file so it looks like this:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```
Remove these elements from AppHost and ServiceDefaults projects.

### 10. Add project references

```bash
# Users project references Users.Contracts
dotnet add Nimble.Modulith.Users\Nimble.Modulith.Users.csproj reference Nimble.Modulith.Users.Contracts\Nimble.Modulith.Users.Contracts.csproj

# Web project references Users
dotnet add Nimble.Modulith.Web\Nimble.Modulith.Web.csproj reference Nimble.Modulith.Users\Nimble.Modulith.Users.csproj

# Web project references ServiceDefaults (if using Aspire)
dotnet add Nimble.Modulith.Web\Nimble.Modulith.Web.csproj reference Nimble.Modulith.ServiceDefaults\Nimble.Modulith.ServiceDefaults.csproj

# AspireHost references ServiceDefaults and Web
dotnet add Nimble.Modulith.AppHost\Nimble.Modulith.AppHost.csproj reference Nimble.Modulith.ServiceDefaults\Nimble.Modulith.ServiceDefaults.csproj
dotnet add Nimble.Modulith.AppHost\Nimble.Modulith.AppHost.csproj reference Nimble.Modulith.Web\Nimble.Modulith.Web.csproj
```

### 11. Update namespaces (optional)

You may want to update the default namespaces in the Users projects:

- Update `Nimble.Modulith.Users\Class1.cs` to use namespace `Nimble.Modulith.Users`
- Update `Nimble.Modulith.Users.Contracts\Class1.cs` to use namespace `Nimble.Modulith.Users.Contracts`

Or simply delete the generated `Class1.cs` files and create your own classes as needed.

### Verify the solution

After completing these steps, you can verify your solution structure:

```bash
dotnet sln list
```

This should show all five projects with the Users projects organized under the "Users Module" solution folder.

## Option 2: Creating the Solution with Ardalis.Modulith templates

Install the modulith dotnet new template:

```bash
dotnet new install Ardalis.Modulith@2.0.0-beta3
```
