<configuration>
  <config>
    <add key="no_proxy" value="*.onefiserv.net" />
    <add key="http_proxy" value="http://proxy-na0.fiserv.one:8080/" />
  </config>
  <packageSources>
    <add key="FlumePublicGroup" value="https://nexus-dev.onefiserv.net/repository/nuget-gl-flume-public-group/index.json" />
    <add key="NugerOrgVer2" value="https://nexus-dev.onefiserv.net/repository/nuget.org-v2" />
    <add key="cs-bp-dotnet" value="https://pkgs.dev.azure.com/fiservcards/Bill-Pay/_packaging/cs-bp-dotnet/nuget/v3/index.json" />
  </packageSources>
  <disabledPackageSources>
    <add key="NugerOrgVer2" value="true" />
  </disabledPackageSources>
  <packageSourceCredentials>
    <FlumePublicGroup>
      <add key="Username" value="o12l6hpv" />
      <add key="ClearTextPassword" value="7fZxpXKb01i2C10JwzZbnrRMa0DaICa8F0mo1XBKNTUq" />
    </FlumePublicGroup>
    <NugerOrgVer2>
      <add key="Username" value="o12l6hpv" />
      <add key="ClearTextPassword" value="7fZxpXKb01i2C10JwzZbnrRMa0DaICa8F0mo1XBKNTUq" />
    </NugerOrgVer2>
  </packageSourceCredentials>
</configuration>
