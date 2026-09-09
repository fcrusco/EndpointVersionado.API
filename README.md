# Web API versionada em C# — Passo a passo

Guia de construção de uma Web API .NET com endpoints versionados (v1/v2), Swagger com múltiplos documentos e marcação de deprecation.

## 1. Criar o projeto

```bash
dotnet new webapi -n ClimaApi --use-controllers
cd ClimaApi
```

## 2. Adicionar os pacotes corretos

```bash
dotnet add package Asp.Versioning.Mvc
dotnet add package Asp.Versioning.Mvc.ApiExplorer
dotnet add package Swashbuckle.AspNetCore
```

> ⚠️ **Não instalar/remover** `Microsoft.AspNetCore.OpenApi` — ele traz o `Microsoft.OpenApi` v2.x, incompatível com o Swashbuckle (remove o namespace `Microsoft.OpenApi.Models`). Se já estiver no projeto:
>
> ```bash
> dotnet remove package Microsoft.AspNetCore.OpenApi
> dotnet clean
> dotnet restore
> ```

## 3. Configurar o `Program.cs`

```csharp
using Asp.Versioning;
using Asp.Versioning.ApiExplorer;
using Microsoft.Extensions.Options;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
})
.AddMvc()
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.ConfigureOptions<ConfigureSwaggerOptions>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        var provider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
        foreach (var description in provider.ApiVersionDescriptions)
        {
            options.SwaggerEndpoint(
                $"/swagger/{description.GroupName}/swagger.json",
                description.GroupName.ToUpperInvariant());
        }
    });
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.Run();

class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions>
{
    private readonly IApiVersionDescriptionProvider _provider;

    public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider)
    {
        _provider = provider;
    }

    public void Configure(SwaggerGenOptions options)
    {
        foreach (var description in _provider.ApiVersionDescriptions)
        {
            options.SwaggerDoc(description.GroupName, new OpenApiInfo
            {
                Title = "Clima API",
                Version = description.ApiVersion.ToString(),
                Description = description.IsDeprecated
                    ? "Esta versão está obsoleta."
                    : null
            });
        }
    }
}
```

## 4. Criar as pastas de versão

```
Controllers/
  v1/
    WeatherForecastController.cs
  v2/
    WeatherForecastController.cs
```

## 5. Mover o controller original para `v1` e marcar como obsoleto

```csharp
[ApiVersion("1.0", Deprecated = true)]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class WeatherForecastController : ControllerBase
{
    [HttpGet]
    [Obsolete("Este endpoint está obsoleto. Utilize a v2, que inclui a sensação térmica.")]
    public IEnumerable<WeatherForecast> Get()
    {
        // lógica original do template
    }
}
```

## 6. Copiar o controller para `v2` e evoluir o contrato

```csharp
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class WeatherForecastController : ControllerBase
{
    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        // mesma lógica + novo campo, ex: FeelsLikeC
    }
}
```

## 7. Rodar e validar

```bash
dotnet build
dotnet run
```

Abrir o Swagger UI e conferir:

- Dropdown com **v1** e **v2**
- **v1** aparece riscado/marcado como obsoleto
- Header `api-supported-versions` na resposta da chamada

## Problemas comuns e soluções

### `ApiVersion`, `UrlSegmentApiVersionReader` não encontrados

Causa: `using Microsoft.AspNetCore.Mvc.Versioning;` (namespace do pacote antigo, descontinuado).

Solução: usar `using Asp.Versioning;` (namespace do pacote atual `Asp.Versioning.Mvc`).

### `AddSwaggerGen` não encontrado

Causa: instalar apenas `Swashbuckle.AspNetCore.SwaggerUI` (só a interface visual), sem o gerador de documentos.

Solução: instalar o meta-pacote `Swashbuckle.AspNetCore`, que já inclui `SwaggerGen` + `Swagger` + `SwaggerUI`.

### `using Microsoft.OpenApi.Models;` não compila

Causa: o pacote `Microsoft.AspNetCore.OpenApi` traz transitivamente o `Microsoft.OpenApi` v2.x, que **removeu o namespace `Microsoft.OpenApi.Models`**. O Swashbuckle espera a v1.6.x, onde esse namespace existe.

Solução:

```bash
dotnet remove package Microsoft.AspNetCore.OpenApi
dotnet clean
dotnet restore
```

Para investigar conflitos de versão transitiva:

```bash
dotnet list package --include-transitive
```

## Estratégias de leitura de versão (alternativas ao `UrlSegmentApiVersionReader`)

| Estratégia | Exemplo | Característica |
|---|---|---|
| `UrlSegmentApiVersionReader` | `/api/v1/...` | Mais visível e "REST-friendly" |
| `QueryStringApiVersionReader` | `/api/...?api-version=1.0` | Fácil de testar no navegador |
| `HeaderApiVersionReader` | `X-Api-Version: 1.0` | Mais limpo, porém menos descobrível |

Também é possível combinar estratégias com `ApiVersionReader.Combine(...)`.

## Caminhos alternativos para explorar em aula

- **Minimal APIs**: o mesmo versionamento funciona com `MapGroup` e `.HasApiVersion(...)`, sem precisar de controllers.
- **Sunset Policy (RFC 8594)**: cabeçalho `Sunset` como padrão de mercado para avisar quando um endpoint efetivamente sairá do ar, complementando o `[Obsolete]`/`Deprecated`.
