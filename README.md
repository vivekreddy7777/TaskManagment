using System.Text;
using System.Xml;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

public sealed class XmlConsumer
{
    private readonly ILogger<XmlConsumer> _log;
    private readonly IServiceScopeFactory _scopeFactory;

    public XmlConsumer(ILogger<XmlConsumer> log, IServiceScopeFactory scopeFactory)
    {
        _log = log;
        _scopeFactory = scopeFactory;
    }

    public async Task ProcessXmlAsync(string xml, CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(xml))
        {
            _log.LogWarning("XML payload empty — skipping.");
            return;
        }

        // 1) Parse Request_Number (non-throwing)
        long? req = TryParseRequestNumber(xml);

        // 2) Persist raw XML for audit/diagnostics
        var work = Path.Combine(Path.GetTempPath(), "BTA_CompareTool",
                                DateTime.UtcNow.ToString("yyyyMMdd_HHmmss_fff"));
        Directory.CreateDirectory(work);
        var xmlPath = Path.Combine(work, req.HasValue ? $"req_{req}.xml" : "payload.xml");
        await File.WriteAllTextAsync(xmlPath, xml, new UTF8Encoding(false), ct);
        _log.LogInformation("Saved payload → {Path}", xmlPath);

        // 3) Build Job exactly like the service does
        Job job = req.HasValue
            ? new Job(req.Value,  module: "CompareTool", table: "CompareClaimsDetails", sp: "procCompareTool_Logic")
            : new Job(xml,        module: "CompareTool", table: "CompareClaimsDetails", sp: "procCompareTool_Logic");

        // 4) Run your processors
        using var scope = _scopeFactory.CreateScope();
        var compareJob = ActivatorUtilities.CreateInstance<BTA_CompareTool_Job>(scope.ServiceProvider, job);

        await compareJob.GetJobDetails(ct);
        await compareJob.LoadInputFile(xmlPath, ct); // keep file-based overload
        await compareJob.CallCompareTool(ct);
        await compareJob.ExportResults(ct);

        _log.LogInformation("CompareTool pipeline completed.");
    }

    private static long? TryParseRequestNumber(string xml)
    {
        try
        {
            var settings = new XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit, IgnoreComments = true, IgnoreWhitespace = true };
            using var xr = XmlReader.Create(new StringReader(xml), settings);
            var doc = new XmlDocument();
            doc.Load(xr);
            var n = doc.SelectSingleNode("/message/Request_Number") ??
                    doc.SelectSingleNode("/Message/Request_Number");
            return (n != null && long.TryParse(n.InnerText.Trim(), out var v)) ? v : null;
        }
        catch { return null; }
    }
}
