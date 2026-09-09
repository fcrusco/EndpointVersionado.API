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
using Asp.Versioning; // Traz ApiVersion e os leitores de versão (UrlSegmentApiVersionReader, etc.)
using Asp.Versioning.ApiExplorer; // Traz IApiVersionDescriptionProvider, usado para listar as versões no Swagger
using Microsoft.Extensions.Options; // Traz IConfigureOptions<T>, usado para configurar o Swagger por versão
using Microsoft.OpenApi;
using Swashbuckle.AspNetCore.SwaggerGen; // Traz SwaggerGenOptions, configurado pela classe auxiliar abaixo

var builder = WebApplication.CreateBuilder(args); // Cria o builder padrão da aplicação (config, DI, logging)

// Versionamento
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0); // Versão usada quando o cliente não informa nenhuma
    options.AssumeDefaultVersionWhenUnspecified = true; // Evita erro 400 se a versão não vier na URL
    options.ReportApiVersions = true; // Adiciona header "api-supported-versions" na resposta
    options.ApiVersionReader = new UrlSegmentApiVersionReader(); // Lê a versão do segmento da URL (ex: /api/v1/...)
})
.AddMvc() // Integra o versionamento com Controllers/MVC (atributos [ApiVersion], rotas, etc.)
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV"; // Formata o nome do grupo Swagger como "v1", "v2"
    options.SubstituteApiVersionInUrl = true; // Substitui {version:apiVersion} pelo valor real na URL do Swagger
});

builder.Services.AddControllers(); // Habilita suporte a Controllers (necessário para [ApiController])
builder.Services.AddEndpointsApiExplorer(); // Habilita a descoberta automática de endpoints para geração de docs
builder.Services.AddSwaggerGen(); // Registra o gerador de documentos Swagger (sem configuração de versão ainda)

// Configura os documentos do Swagger dinamicamente por versão
builder.Services.ConfigureOptions<ConfigureSwaggerOptions>(); // Plugga a classe abaixo no pipeline de opções do SwaggerGen

var app = builder.Build(); // Constrói a aplicação e finaliza o container de injeção de dependência

if (app.Environment.IsDevelopment()) // Só expõe o Swagger em ambiente de desenvolvimento
{
    app.UseSwagger(); // Gera o JSON do OpenAPI em /swagger/{versao}/swagger.json
    app.UseSwaggerUI(options =>
    {
        var provider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>(); // Pega as versões já registradas
        foreach (var description in provider.ApiVersionDescriptions) // Uma iteração por versão descoberta (v1, v2...)
        {
            options.SwaggerEndpoint(
                $"/swagger/{description.GroupName}/swagger.json", // Caminho do JSON daquela versão
                description.GroupName.ToUpperInvariant()); // Texto do dropdown na UI (ex: "V1", "V2")
        }
    });
}

app.UseHttpsRedirection(); // Força redirecionamento HTTP -> HTTPS
app.UseAuthorization(); // Habilita middleware de autorização (mesmo sem regras definidas ainda)
app.MapControllers(); // Mapeia as rotas dos Controllers decorados com [ApiController]
app.Run(); // Inicia o servidor e bloqueia a thread principal

// Registrar um SwaggerDoc por versão de API
class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions> // Implementa a interface que o DI chama automaticamente
{
    private readonly IApiVersionDescriptionProvider _provider; // Guarda a referência das versões disponíveis

    public ConfigureSwaggerOptions(IApiVersionDescriptionProvider provider) // Injetado automaticamente pelo container
    {
        _provider = provider; // Armazena para uso no método Configure
    }

    public void Configure(SwaggerGenOptions options) // Chamado pelo Swashbuckle antes de gerar os documentos
    {
        foreach (var description in _provider.ApiVersionDescriptions) // Um SwaggerDoc por versão registrada
        {
            options.SwaggerDoc(description.GroupName, new OpenApiInfo // Nome do grupo (ex: "v1") vira o nome do doc
            {
                Title = "Clima API", // Título exibido na UI do Swagger
                Version = description.ApiVersion.ToString(), // Versão exibida (ex: "1.0")
                Description = description.IsDeprecated // Mostra aviso somente se a versão estiver marcada como obsoleta
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


- **Sunset Policy (RFC 8594)**: cabeçalho `Sunset` como padrão de mercado para avisar quando um endpoint efetivamente sairá do ar, complementando o `[Obsolete]`/`Deprecated`.
