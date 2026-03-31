FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

WORKDIR /src
COPY . .
RUN dotnet publish CoreCms.Net.Web.WebApi/CoreCms.Net.Web.WebApi.csproj -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:9.0

WORKDIR /app

EXPOSE 80

COPY --from=build /app/publish .

ENV TZ=Asia/Shanghai

RUN sed -i 's/MinProtocol = TLSv1.2/MinProtocol = TLSv1/g' /etc/ssl/openssl.cnf || true
RUN sed -i 's/MinProtocol = TLSv1.2/MinProtocol = TLSv1/g' /usr/lib/ssl/openssl.cnf || true

ENTRYPOINT ["dotnet", "CoreCms.Net.Web.WebApi.dll"]