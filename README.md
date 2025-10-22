using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Serilog;
using Serilog.Events;
using System;
using System.IO;
using System.Threading.Tasks;
using System.Xml.Linq;

namespace CompareTool.ConsoleApp
{
    public class Program
    {
        public static async Task Main(string[] args)
        {
            // Bootstrap Serilog
            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                .WriteTo.Console()
                .WriteTo.File("Logs/CompareToolConsole.log", rollingInterval: RollingInterval.Day)
                .CreateLogger();

            try
            {
                Log.Information("==== CompareTool Console Started ====");

                // Build host with configuration and EF
                var host = Host.CreateDefaultBuilder(args)
                    .UseSerilog()
                    .ConfigureAppConfiguration(cfg =>
                    {
                        cfg.AddJsonFile("appsettings.json", optional: true)
                           .AddEnvironmentVariables();
                    })
                    .ConfigureServices((ctx, services) =>
                    {
                        services.AddDbContext<CompareToolDbContext>(opt =>
                            opt.UseSqlServer(ctx.Configuration.GetConnectionString("CompareToolDb")));
                        services.AddScoped<XmlConsumer>();
                    })
                    .Build();

                // Resolve consumer and process XML
                using var scope = host.Services.CreateScope();
                var consumer = scope.ServiceProvider.GetRequiredService<XmlConsumer>();

                string xml = GetArgumentValue(args, "-xml") ?? string.Empty;
                if (string.IsNullOrWhiteSpace(xml))
                {
                    Log.Warning("No XML payload received from service.");
                    return;
                }

                await consumer.ProcessXmlAsync(xml);
                Log.Information("==== XML processing completed successfully ====");
            }
            catch (Exception ex)
            {
                Log.Fatal(ex, "Unhandled exception while processing XML.");
            }
            finally
            {
                Log.CloseAndFlush();
            }
        }

        private static string? GetArgumentValue(string[] args, string key)
        {
            int index = Array.IndexOf(args, key);
            return (index >= 0 && index + 1 < args.Length) ? args[index + 1] : null;
        }
    }

    // Handles XML persistence
    public class XmlConsumer
    {
        private readonly CompareToolDbContext _db;

        public XmlConsumer(CompareToolDbContext db) => _db = db;

        public async Task ProcessXmlAsync(string xml)
        {
            try
            {
                // Validate XML format
                XDocument.Parse(xml);

                // Save XML payload
                var entity = new PayloadLog
                {
                    XmlPayload = xml,
                    ReceivedAt = DateTime.UtcNow
                };

                _db.PayloadLogs.Add(entity);
                await _db.SaveChangesAsync();

                Log.Information("XML saved successfully (Id: {Id})", entity.Id);
            }
            catch (Exception ex)
            {
                Log.Error(ex, "Failed to process or save XML payload.");
            }
        }
    }

    // EF Core DbContext
    public class CompareToolDbContext : DbContext
    {
        public CompareToolDbContext(DbContextOptions<CompareToolDbContext> options)
            : base(options) { }

        public DbSet<PayloadLog> PayloadLogs => Set<PayloadLog>();
    }

    // EF entity
    public class PayloadLog
    {
        public int Id { get; set; }
        public string XmlPayload { get; set; } = string.Empty;
        public DateTime ReceivedAt { get; set; }
    }
}
