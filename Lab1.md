# Lab 1: Create a modular monolith solution with one module for Users

In this lab we will create the initial structure of the modular monolith application. We will mainly use the `dotnet` CLI tool, and you can script the setup once you're comfortable with it to suit your own team/org needs and standards.

When we're done, we will have a solution file called `Nimble.Modulith.slnx` which will contain five projects:

- AspireHost
- ServiceDefaults
- Web (ASP.NET Core app)
- Users
- Users.Contracts

## Creating the Solution with dotnet CLI

Follow these steps to create the modular monolith solution from scratch. You can do this in a folder called `labs` (which is in the .gitignore file and so won't be added to git by default). If you want source control, create a new repo or use a subfolder called `mylabs` which should be tracked by git.

### 1. Create the AspireHost project

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

### 2. Change to slnx file

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

### 3. Create the ASP.NET Core minimal Web API project

Run the following commands from the same folder as your slnx file.

```bash
dotnet new webapi --name Nimble.Modulith.Web --output Nimble.Modulith.Web --use-minimal-apis --framework net10.0
dotnet sln add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj --solution-folder "_Host"
```

### 4. Create the Users class library

```bash
dotnet new classlib --name Nimble.Modulith.Users --output Nimble.Modulith.Users --framework net10.0
```

### 5. Create solution folder for Users Module

```bash
dotnet sln add Nimble.Modulith.Users/Nimble.Modulith.Users.csproj --solution-folder "Users Module"
```

### 6. Create the Users.Contracts class library

```bash
dotnet new classlib --name Nimble.Modulith.Users.Contracts --output Nimble.Modulith.Users.Contracts --framework net10.0
dotnet sln add Nimble.Modulith.Users.Contracts/Nimble.Modulith.Users.Contracts.csproj --solution-folder "Users Module"
```

### 7. Verify all projects are in the solution file

```bash
dotnet sln list
```

You should see all 5 projects. If not you can use `dotnet sln add <csproj path>` to add any that are missing.

Move the Aspire projects to the `_Host` solution folder:

```bash
dotnet sln remove Nimble.Modulith.AppHost/Nimble.Modulith.AppHost.csproj
dotnet sln add Nimble.Modulith.AppHost/Nimble.Modulith.AppHost.csproj --solution-folder "_Host"
dotnet sln remove Nimble.Modulith.ServiceDefaults/Nimble.Modulith.ServiceDefaults.csproj
dotnet sln add Nimble.Modulith.ServiceDefaults/Nimble.Modulith.ServiceDefaults.csproj --solution-folder "_Host"
```

This will change the default startup project. When you open in an IDE you'll need to set AppHost as the startup project manually.

### 8. Use Central Package Management

We're going to have a lot of projects. We don't want to have different package versions all over the place. Run this from the folder with the slnx file. We're going to use this tool to help: [CentralizedPackageConverter](https://github.com/Webreaper/CentralisedPackageConverter)

```bash
 dotnet tool install -g CentralisedPackageConverter
 central-pkg-converter .
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

### 12. Add Directory files to Solution Folder

Now we'll make sure we can see the Directory.Build.props and Directory.Packages.props files in our solution in VS2026 (or your IDE of choice). Open the Nimble.Modulith.slnx file and add this at the top:

```xml
<Folder Name="/__SolutionItems/">
  <File Path="Directory.Build.props" />
  <File Path="Directory.Packages.props" />
</Folder>
```

The whole thing should look like this:

```xml
<Solution>
  <Folder Name="/__SolutionItems/">
    <File Path="Directory.Build.props" />
    <File Path="Directory.Packages.props" />
  </Folder>
  <Folder Name="/Users Module/">
    <Project Path="Nimble.Modulith.Users.Contracts/Nimble.Modulith.Users.Contracts.csproj" />
    <Project Path="Nimble.Modulith.Users/Nimble.Modulith.Users.csproj" />
  </Folder>
  <Folder Name="/_Host/">
    <Project Path="Nimble.Modulith.AppHost/Nimble.Modulith.AppHost.csproj" />
    <Project Path="Nimble.Modulith.ServiceDefaults/Nimble.Modulith.ServiceDefaults.csproj" />
    <Project Path="Nimble.Modulith.Web/Nimble.Modulith.Web.csproj" />
  </Folder>
</Solution>
```

## Summary

At this point we have the basics: 2 Aspire projects, a eb host project, and 2 projects that represent our Users module. In the next lab we will configure Aspire to work with the web host and module and get user registration working.
