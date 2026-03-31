FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

WORKDIR /src
COPY . .
RUN dotnet publish CoreCms.Net.Web.Admin/CoreCms.Net.Web.Admin.csproj -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:9.0

RUN apt-get update && apt-get install -y libgdiplus && rm -rf /var/lib/apt/lists/*

WORKDIR /app

EXPOSE 80

COPY --from=build /app/publish .

ENV TZ=Asia/Shanghai

ENTRYPOINT ["dotnet", "CoreCms.Net.Web.Admin.dll"]