using System;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using BTA; // Make sure this matches your namespace (same as Job.cs / BTA_CompareTool_Job)
using System.Text;

namespace BTA_CompareTool_Console
{
    public class XmlConsumer
    {
        private readonly ILogger<XmlConsumer> _log;
        private readonly IServiceScopeFactory _scopeFactory;

        public XmlConsumer(ILogger<XmlConsumer> log, IServiceScopeFactory scopeFactory)
        {
            _log = log;
            _scopeFactory = scopeFactory;
        }

        // 👇 Your main method — this is called from Program.cs
        public async Task ProcessXmlAsync(string xml, CancellationToken ct = default)
        {
            _log.LogInformation("Received XML payload from service: {xml}", xml);

            // 1️⃣ Parse RequestNumber (optional)
            long? requestNumber = null;
            if (long.TryParse(xml, out var parsed))
                requestNumber = parsed;

            // 2️⃣ Build Job just like the service
            Job job = requestNumber.HasValue
                ? new Job(requestNumber.Value, "CompareTool", "CompareClaimsDetails", "procCompareTool_Logic")
                : new Job(xml, "CompareTool", "CompareClaimsDetails", "procCompareTool_Logic");

            // 3️⃣ Resolve BTA_CompareTool_Job from DI
            using var scope = _scopeFactory.CreateScope();
            var compareJob = ActivatorUtilities.CreateInstance<BTA_CompareTool_Job>(
                scope.ServiceProvider, job);

            // 4️⃣ Run job pipeline
            await compareJob.GetJobDetails(ct);
            await compareJob.LoadInputFile(ct);
            await compareJob.CallCompareTool(ct);
            await compareJob.ExportResults(ct);

            _log.LogInformation("CompareTool console job completed successfully.");
        }
    }
}
