public async Task ProcessXmlAsync(string xml, CancellationToken ct = default)
{
    _log.LogInformation("Received XML from service: {Xml}", xml);

    // Parse request number (optional)
    long? requestNumber = null;
    if (long.TryParse(xml, out var parsed))
        requestNumber = parsed;

    // Build job just like the service does
    Job job = requestNumber.HasValue
        ? new Job(requestNumber.Value, "CompareTool", "CompareClaimsDetails", "procCompareTool_Logic")
        : new Job(xml, "CompareTool", "CompareClaimsDetails", "procCompareTool_Logic");

    using var scope = _scopeFactory.CreateScope();
    var compareJob = ActivatorUtilities.CreateInstance<BTA_CompareTool_Job>(scope.ServiceProvider, job);

    await compareJob.GetJobDetails(ct);
    await compareJob.LoadInputFile(ct);
    await compareJob.CallCompareTool(ct);
    await compareJob.ExportResults(ct);

    _log.LogInformation("CompareTool job completed successfully.");
}
