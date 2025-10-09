using System.Net.Http.Json;
using BTA_CompareTool_Service;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using Serilog.Events;

public static class Program
{
    public static async Task Main(string[] args)
    {
        // --- Bootstrap Serilog early (before host) ---
        Directory.CreateDirectory(Path.Combine(AppContext.BaseDirectory, "Logs"));
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Information()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.File(Path.Combine(AppContext.BaseDirectory, "Logs", "service-boot.log"),
                          rollingInterval: RollingInterval.Day, retainedFileCountLimit: 14)
            .CreateLogger();

        try
        {
            Log.Information("=== CompareTool service bootstrap ===");

            // 1) base config: local JSON + env
            var env = Environment.GetEnvironmentVariable("DOTNET_ENVIRONMENT") ?? "Production";
            var baseCfg = new ConfigurationBuilder()
                .SetBasePath(AppContext.BaseDirectory)
                .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true)
                .AddEnvironmentVariables()
                .Build();

            // 2) pull key/values from your config server (if configured)
            var configServerUrl = baseCfg["ConfigServerPath"];
            var serverValues = configServerUrl is { Length: > 0 }
                ? await ConfigServerClient.FetchAsync(configServerUrl, Log.Logger)
                : new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);

            if (serverValues.Count == 0 && !string.IsNullOrWhiteSpace(configServerUrl))
                Log.Warning("Config server returned no values from {Url}", configServerUrl);

            // (optional) push into legacy AppSettings for any old callers
            foreach (var kv in serverValues)
                System.Configuration.ConfigurationManager.AppSettings.Set(kv.Key, kv.Value);

            // 3) FINAL merged configuration (server overrides JSON/env)
            var mergedConfig = new ConfigurationBuilder()
                .AddConfiguration(baseCfg)
                .AddInMemoryCollection(serverValues) // last wins
                .Build();

            // 4) sanity check required connection string
            var dbCs = mergedConfig.GetConnectionString("BTEBDConnectionString");
            if (string.IsNullOrWhiteSpace(dbCs))
                throw new InvalidOperationException("ConnectionStrings:BTEBDConnectionString missing.");

            // 5) Build the worker host
            var host = Host.CreateDefaultBuilder(args)
                .UseWindowsService() // comment this out when running as console on Linux
                .UseSerilog((ctx, cfg) =>
                {
                    cfg.ReadFrom.Configuration(mergedConfig)
                       .Enrich.FromLogContext()
                       .WriteTo.Console()
                       .WriteTo.File(Path.Combine(AppContext.BaseDirectory, "Logs", "service.log"),
                                     rollingInterval: RollingInterval.Day, retainedFileCountLimit: 14);

                    if (!Environment.UserInteractive)
                    {
                        // requires Serilog.Sinks.EventLog package
                        cfg.WriteTo.EventLog("BTA_CompareTool_Service", manageEventSource: true,
                            restrictedToMinimumLevel: LogEventLevel.Error);
                    }
                })
                .ConfigureServices(services =>
                {
                    // Typed options from CompareTool section
                    services.AddOptions<CompareToolOptions>()
                        .Bind(mergedConfig.GetSection("CompareTool"))
                        .Configure<IConfiguration>((o, cfg) =>
                        {
                            // Connection string comes from standard ConnectionStrings section
                            o.ConnectionString = cfg.GetConnectionString("BTEBDConnectionString") ?? string.Empty;
                        })
                        .Validate(o => !string.IsNullOrWhiteSpace(o.ConnectionString), "Connection string required.")
                        .Validate(o => !string.IsNullOrWhiteSpace(o.Module), "Module required.")
                        .ValidateOnStart();

                    // Hosted service
                    services.AddHostedService<CompareToolHostedService>();
                })
                .Build();

            Log.Information("CompareTool service starting...");
            await host.RunAsync();
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "Service failed to start.");
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }
}

/// <summary>
/// Minimal client that reads your config server and returns a flat key/value dictionary.
/// It accepts two common shapes:
/// 1) { "key":"value", "key2":"value2" }
/// 2) [ { "name":"key", "value":"value" }, ... ]
/// </summary>
internal static class ConfigServerClient
{
    public static async Task<Dictionary<string, string>> FetchAsync(string url, ILogger logger)
    {
        using var http = new HttpClient { Timeout = TimeSpan.FromSeconds(15) };

        try
        {
            using var resp = await http.GetAsync(url);
            resp.EnsureSuccessStatusCode();

            // Try map 1: simple dictionary
            try
            {
                var dict = await resp.Content.ReadFromJsonAsync<Dictionary<string, string>>();
                if (dict is { Count: > 0 })
                    return new Dictionary<string, string>(dict, StringComparer.OrdinalIgnoreCase);
            }
            catch { /* fall through */ }

            // Try map 2: array of { name, value }
            var items = await resp.Content.ReadFromJsonAsync<List<NameValueItem>>();
            var result = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
            if (items != null)
                foreach (var i in items)
                    if (!string.IsNullOrWhiteSpace(i.name))
                        result[i.name] = i.value ?? string.Empty;

            return result;
        }
        catch (Exception ex)
        {
            logger.Error(ex, "Failed to fetch config from {Url}", url);
            return new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);
        }
    }

    private sealed record NameValueItem(string name, string? value);
}
